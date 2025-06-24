---
title: Game from Scratch 002 - Fighting Windows
description: Building a game from scratch with only C. Creating a window and handling blocking during drag and resize of window.
tags:
    - game
    - game dev
    - c
    - window creation
    - message loop
    - modal loop
date: 2025-06-24
---

[Last time]({{< relref "posts/game-from-scratch-001-choosing-c.md" >}}), I said I'd create a window and handle the Windows message loop and that's what I did! Before diving into the code, I want to clarify something: this blog series won't be a step-by-step tutorial on building a game from scratch in C. Instead, I'll share my progress and focus on interesting problems and how I solve them.

## The Bug
Right now, I have an awesome looking game: a square bouncing back and forth. But there's one tiny problem stopping me from selling a million copies. Check out the gif below, can you spot it?

![Game freezes when resizing or moving the window](/images/game-before-fix.gif)

Pretty obvious, right? The game freezes when you resize or drag the window. You might call it an edge case, but it bugged me enough to investigate. I checked games made with Godot and SDL and they don't freeze, so I had to be missing something.

### The Message Loop
First, some background. Windows communicates with applications using messages: keystrokes, window movement, close requests, etc. These come in as numeric codes with metadata attached, and it's up to us to handle them using a message loop.

Here’s my setup:

```c
void frame(void)
{
    MSG msg;
    //Checks if we have any incoming messages in our message queue and retrieves them if any exist.
    while(PeekMessageW(&msg, 0, 0, 0, PM_REMOVE))
    {
        if(msg.message == WM_QUIT)
        {
            isRunning = 0;
        }
        else 
        {
            //Translate to actual characters
            TranslateMessage(&msg);

            //Send to the window procedure
            DispatchMessageW(&msg);
        }
    }

    //Code for rendering a moving square here...
}

//This is what gets called in the end after we call DispatchMessageW in the frame function
static LRESULT CALLBACK WindowProc(HWND window, UINT message, WPARAM wParam, LPARAM lParam)
{
    LRESULT result = 0;
    switch(message)
    {
        case WM_CLOSE:
        case WM_DESTROY:
        {
            //Ends up as a WM_QUIT which we handle in our frame function
            PostQuitMessage(0);
        } break;
        default:
        {
            result = DefWindowProcW(window, message, wParam, lParam);
        } break;
    }

    return result;
}
```
This works well, except when the user drags or resizes the window.

## Why It Freezes
When you drag or resize a window, Windows enters an internal modal loop that blocks the thread that created the window. I'm not entirely sure why but I think it's to handle mouse interaction. But the key point is: the application freezes which isn't great for a game.

## The Fix
One option is to run rendering on a separate thread. That works, but it can add a frame of latency, and I don’t want that for this project.

Instead, I used the same trick that Godot and SDL uses. When Windows enters a modal loop, it sends a message. You can catch that and set a timer that fires regularly. Each time the timer triggers, Windows sends a WM_TIMER message where I call my frame function. When the modal loop ends, I kill the timer.

It feels like a hack, but it’s simple and it works.
```c
static LRESULT CALLBACK WindowProc(HWND window, UINT message, WPARAM wParam, LPARAM lParam)
{
    LRESULT result = 0;
    switch(message)
    {
        case WM_ENTERSIZEMOVE:
        case WM_ENTERMENULOOP:
        {
            SetTimer(hwnd, TIMER_ID_MOVE_FRAME, USER_TIMER_MINIMUM, NULL);
        } break;
        case WM_EXITSIZEMOVE:
        case WM_EXITMENULOOP:
        {
            KillTimer(hwnd, TIMER_ID_MOVE_FRAME);
        } break;
        case WM_TIMER:
        {
            if(wParam == TIMER_ID_MOVE_FRAME)
            {
                frame();
            }
        } break;
        //...
    }

    return result;
}
```

And just like that, the game no longer freezes when dragging or resizing the window:

![Game no longer freezes when resizing or moving the window](/images/game-after-fix.gif)

## What’s Next?
Not sure yet! It might be input, DirectX, or something else entirely. I’ll keep it a surprise.

