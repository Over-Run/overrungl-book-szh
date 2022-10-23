# 游戏循环

## 理论

在本章中，我们将为游戏引擎创建一个游戏循环。利用游戏循环，我们可在每一帧更新状态，渲染等。

这段伪代码显示了游戏循环基本逻辑。

```c
while (running) {
    update();
    render();
    post_processing();
}
```

然而，这个循环只能处理固定更新与渲染。如果系统运行速度过快或过慢，你将无法观察到运行情况。因此，我们需要使用 **_Timer_**
来计算两次运行时间的间隔（也称 _delta_）。

我们可以使用这个简易 _Timer_。它能计算 _delta_ 并用于更新时的因数。虽然它不能使每秒更新次数相同，但是对现在来说够用了。

```c
double lastTime, currentTime, delta;
while (running) {
    lastTime = currentTime;
    currentTime = getCurrentTime();
    delta = currentTime - lastTime;
    update(delta);
    render();
    post_processing();
}
```

## 实践

了解游戏循环运行原理后，我们就能正式开始编写了。

为了运行游戏，我们需要提供一个接口。这个接口包含基本逻辑，以便我们今后复用。

```java
package io.github.overrunglbook.game;

public interface IGameLogic extends AutoCloseable {
    void init();

    void onKey(int key, int scancode, int action, int mods);

    void onResize(int oldWidth, int oldHeight,
                  int newWidth, int newHeight);

    void update();

    void render();
}
```

我们还会提供一个窗口类封装。

```java
package io.github.overrunglbook.render;

import org.overrun.glib.glfw.Callbacks;
import org.overrun.glib.glfw.GLFW;
import org.overrun.glib.util.ValueInt2;

import java.lang.foreign.MemoryAddress;

public final class Window {
    private final MemoryAddress window;
    private int fbWidth, fbHeight;

    public Window(IGameLogic logic, int width, int height, String title) {
        window = GLFW.createWindow(width, height, title, MemoryAddress.NULL, MemoryAddress.NULL);
        if (window == MemoryAddress.NULL) {
            throw new IllegalStateException("Failed to create the GLFW window");
        }
        var size = GLFW.getFramebufferSize(window);
        fbWidth = size.x();
        fbHeight = size.y();
        GLFW.setKeyCallback(window, (handle, key, scancode, action, mods) ->
            logic.onKey(key, scancode, action, mods));
        GLFW.setFramebufferSizeCallback(window, (handle, w, h) -> {
            logic.onResize(fbWidth, fbHeight, w, h);
            fbWidth = w;
            fbHeight = h;
        });
    }

    public ValueInt2 getSize() {
        return GLFW.getWindowSize(window);
    }

    public void setPos(int x, int y) {
        GLFW.setWindowPos(window, x, y);
    }

    public void makeContextCurrent() {
        GLFW.makeContextCurrent(window);
    }

    public void show() {
        GLFW.showWindow(window);
    }

    public void setShouldClose(boolean value) {
        GLFW.setWindowShouldClose(window, value);
    }

    public void swapBuffers() {
        GLFW.swapBuffers(window);
    }

    public void destroy() {
        Callbacks.free(window);
        GLFW.destroyWindow(window);
    }

    /**
     * returns the window
     *
     * @return the window
     */
    public MemoryAddress window() {
        return window;
    }

    /**
     * returns width of the framebuffer
     *
     * @return width of the framebuffer
     */
    public int fbWidth() {
        return fbWidth;
    }

    /**
     * returns height of the framebuffer
     *
     * @return height of the framebuffer
     */
    public int fbHeight() {
        return fbHeight;
    }
}
```

最后，只需要提供游戏实现并将之前的代码搬过来就行了。

```java
package io.github.overrunglbook.game;

import org.overrun.glib.gl.GL;
import org.overrun.glib.gl.GLConstC;
import org.overrun.glib.glfw.GLFW;

import static org.overrun.glib.gl.GLConstC.*;

public class DummyGame implements IGameLogic {
    private final Window window;

    public void start() {
        GLFWErrorCallback.createPrint().set();

        if (!GLFW.init()) {
            throw new IllegalStateException("Unable to initialize GLFW");
        }

        GLFW.defaultWindowHints();
        GLFW.windowHint(GLFW.VISIBLE, false);
        GLFW.windowHint(GLFW.RESIZABLE, true);

        window = new Window(this, 800, 600, "Hello world");

        var vidMode = GLFW.getVideoMode(GLFW.getPrimaryMonitor());
        if (vidMode != null) {
            var size = window.getSize();
            window.setPos(
                (vidMode.width() - size.x()) / 2,
                (vidMode.height() - size.y()) / 2
            );
        }

        window.makeContextCurrent();
        GLFW.swapInterval(1);

        window.show();
    }

    @Override
    public void init() {
        if (GLCaps.load(GLFW::getProcAddress) == 0)
            throw new IllegalStateException("Failed to load OpenGL");

        GL.clearColor(0.4f, 0.6f, 0.9f, 1.0f);
    }

    @Override
    public void onKey(int key, int scancode, int action, int mods) {
        if (key == GLFW.KEY_ESCAPE && action == GLFW.RELEASE) {
            window.setShouldClose(true);
        }
    }

    @Override
    public void onResize(int oldWidth, int oldHeight,
                         int newWidth, int newHeight) {
        GL.viewport(0, 0, newWidth, newHeight);
    }

    @Override
    public void update() {
    }

    @Override
    public void render() {
        GL.clear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT);
        window.swapBuffers();
        window.pollEvents();
    }

    @Override
    public void close() {
        window.destroy();

        GLFW.terminate();
        GLFW.setErrorCallback(null);
    }
}
```

```java
package io.github.overrunglbook;

public final class Main {
    public static void main(String[] args) {
        try (var game = new DummyGame()) {
            game.start();
        }
    }
}
```
