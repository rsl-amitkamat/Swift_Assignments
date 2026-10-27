# Basics of an iOS Application: Assignment Answers

## 1. AppDelegate and SceneDelegate methods

### a. App launch

AppDelegate:
1. application:willFinishLaunchingWithOptions: 
2. application:didFinishLaunchingWithOptions:
3. application:configurationForConnectingSceneSession:options:

SceneDelegate:
4. scene:willConnectToSession:options:
5. sceneWillEnterForeground:
6. sceneDidBecomeActive:

So the app goes from Not running, to Inactive, to Active. In an app without scenes it would simply be didFinishLaunchingWithOptions followed by applicationDidBecomeActive.

### b. App background (home button)

SceneDelegate:
1. sceneWillResignActive:
2. sceneDidEnterBackground:

AppDelegate: nothing in a scene-based app.

When the user comes back to the app: sceneWillEnterForeground, then sceneDidBecomeActive.

### c. Phone locked (simulator lock button)

Same as pressing the home button:
1. sceneWillResignActive:
2. sceneDidEnterBackground:

AppDelegate gets nothing for the lifecycle itself. On a real device, if the app uses protected files, applicationProtectedDataWillBecomeUnavailable: can also show up in AppDelegate around locking, but it isn't a regular lifecycle callback. Unlocking and returning to the app gives sceneWillEnterForeground followed by sceneDidBecomeActive.

### d. Simulate memory warning

AppDelegate:
1. applicationDidReceiveMemoryWarning:

SceneDelegate: nothing. There's no scene method for memory warnings.

The app stays in the foreground and stays Active the whole time, so none of the scene state methods fire. The warning is just a message to free whatever memory you can (caches, images, anything you can recreate). If the app doesn't free enough, the system may kill it later.

### e. App kill

This one is a bit different from the others because it depends on what state the app is in when it's killed.

When I swipe the app away in the recent apps menu, the sample logs in the training material show:
1. sceneDidDisconnect:

This is where you'd release shared resources and save user data for that scene. AppDelegate's applicationWillTerminate: is only called if the app happens to be running (foreground or background and executing code) when it's terminated. It is not guaranteed, so critical data shouldn't be saved only there.

If the app was already Suspended, the system doesn't wake it up to tell it, so no method is called at all. The same goes when the system purges a suspended app for memory, and when stopping the app using the Stop button in Xcode.

---

## 2. UIViewController methods

### a. App launch

1. viewDidLoad
2. viewWillAppear:
3. viewDidAppear:

viewDidLoad runs once, as the view is loaded (around the time the scene connects and the window is set up). viewWillAppear and viewDidAppear come when the view is about to be, and then actually is, on screen. You'll usually also see viewWillLayoutSubviews and viewDidLayoutSubviews between willAppear and didAppear if you've added logs for those.

### b. App background (home button)

None of the five lifecycle methods are called. This surprised me a bit at first, but it makes sense: viewWillDisappear and viewDidDisappear are about the view controller moving off screen inside the app (navigating, dismissing, switching tabs), not about the whole app being minimized. As far as the view controller knows, it's still the visible screen. If you need to react to the app going to the background from inside a view controller, observe UIApplicationDidEnterBackgroundNotification (or the scene version of it), or use the scene delegate.

Coming back to the foreground also calls nothing (no viewWillAppear). The system may call layout or trait related methods, but not the appear/disappear ones.

### c. Phone locked

Same as b: no appear/disappear methods are called, and nothing on unlock either.

### d. Simulate memory warning

1. didReceiveMemoryWarning

Only this one. The view stays on screen and the view isn't unloaded (that used to happen before iOS 6, with viewDidUnload, which is gone now). Use it to release things you can recreate later.

### e. App kill

Nothing. The process is simply ended, so no viewWillDisappear, viewDidDisappear or even dealloc is called on the view controller. Anything that needs saving has to be saved earlier, for example in sceneDidEnterBackground.

---

## 3. viewWillTransitionToSize: and rotation

The method (Swift name: viewWillTransition(to:with:)) is:

```objc
- (void)viewWillTransitionToSize:(CGSize)size
       withTransitionCoordinator:(id<UIViewControllerTransitionCoordinator>)coordinator {
    [super viewWillTransitionToSize:size withTransitionCoordinator:coordinator];
    NSLog(@"# viewWillTransitionToSize: new size = %@, current size = %@",
          NSStringFromCGSize(size),
          NSStringFromCGSize(self.view.bounds.size));
}
```

What I'd expect to see on an iPhone 11 simulator (414 x 896 points in portrait), rotating with Cmd + left or right arrow:

- Portrait to landscape: size = {896, 414}, while self.view.bounds.size at that moment is still {414, 896}.
- Landscape back to portrait: size = {414, 896}, while the current view bounds are still {896, 414}.

Things I noticed:

- The size parameter is the size the view is going to have after the rotation, not the size it has now. The old size is still available in self.view.bounds at that point, which is why the log is useful.
- It's called before the rotation animation, once per rotation, and the layout methods (viewWillLayoutSubviews and so on) come afterwards with the new size.
- The second parameter is the transition coordinator. It tells you how long the animation takes (around 0.4 seconds) and lets you add your own animations alongside the rotation with animateAlongsideTransition:completion:.
- The size is the view's size, not the device screen size, so if the view controller sits inside a navigation controller or tab bar the numbers are still for the view controller's view.
- Rotating between the two landscape orientations (left to right), or portrait to upside down, doesn't change the width and height, so I wouldn't expect this method to fire for those. Worth checking in the simulator.
- Calling super matters here, same as the other lifecycle methods.

---

## 4. Mapping app states to the callbacks

**Not running to Inactive (launching)**
- AppDelegate: application:didFinishLaunchingWithOptions: (and willFinishLaunching if implemented), application:configurationForConnectingSceneSession:options:
- SceneDelegate: scene:willConnectToSession:options:, sceneWillEnterForeground:
- ViewController: viewDidLoad, viewWillAppear:

**Inactive to Active (foreground, receiving events)**
- SceneDelegate: sceneDidBecomeActive: (old style: applicationDidBecomeActive)
- ViewController: viewDidAppear:

The app is Active once these are done. Everything the user can touch happens in this state.

**Active to Inactive (foreground but not receiving events)**
- SceneDelegate: sceneWillResignActive: (old style: applicationWillResignActive)
- This happens for an incoming call, opening the recent apps menu or Control Center, or just before the app moves to the background. No ViewController callbacks.

**Inactive to Background**
- SceneDelegate: sceneDidEnterBackground: (old style: applicationDidEnterBackground)
- This is the place to save data and release shared resources. No ViewController callbacks.

**Background to Foreground (coming back)**
- SceneDelegate: sceneWillEnterForeground:, then sceneDidBecomeActive:
- This is where state gets restored. No ViewController callbacks.

**Background to Suspended**
- Nothing. The system does this on its own and doesn't tell the app. The app stays in memory but runs no code.

**Suspended or Background to Not running**
- SceneDelegate: sceneDidDisconnect: when the scene is released (this is what shows up in the sample logs when the app is swiped away)
- AppDelegate: applicationWillTerminate: only if the app was actually running at the time, not guaranteed
- If the app was suspended, no callbacks. The system can purge it any time without notice.

**Any running state**
- AppDelegate: applicationDidReceiveMemoryWarning: and ViewController: didReceiveMemoryWarning. These don't change the state, they just ask the app to free memory.

**While Active, inside the app only**
- ViewController: viewWillDisappear: and viewDidDisappear: are called when the screen goes away within the app (navigating to another screen, dismissing it). They're tied to the screen, not to the app state.
