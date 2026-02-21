---
title: "Brownfield React Native app with multiple screens"
description: "Integrating React Native into existing app"
pubDate: "February 21 2026"
---

We are currently deep in the process of integrating React Native code into existing iOS and Android codebases (so-called brownfield integration).

We mainly use React Navigation inside our RN application, but sometimes there is a need to display an RN screen using native navigation. For example, one of our RN screens is displayed in a tab bar. However, screens opened from within this screen should appear above the tab bar. Therefore, we couldn’t find a better solution than opening a new RN screen natively.

React Native and Expo provide good documentation about integrating React Native into native codebases, but I have the feeling that they mainly focus on scenarios where you never open another RN screen on top of an already displayed one.

After several days of debugging, I finally found a solution for both platforms.

## Android

We use fragments to display RN screens inside our native Android application. We followed the instructions exactly as described in the [documentation](https://reactnative.dev/docs/next/integration-with-android-fragment).

However, sometimes when QA tried to open a nested RN screen, they would get a blank (white) screen instead.

It turns out there is something like a race condition in the `ReactFragment` code. After opening the second screen, the first screen’s fragment invokes `onPause`, which in turn calls `reactHost.onPause`. This prevents the second screen from displaying. I left a [comment](https://github.com/facebook/react-native/issues/41850#issuecomment-3937642285) on the related bug thread in the React Native GitHub repository, and hopefully it will be resolved one day.

In the meantime, you can simply create your own fragment with the problematic lifecycle lines (`onPause`, `onDestroy`) commented out or removed:

```kotlin
package <your package name>

import android.content.Intent
import android.os.Bundle
import android.view.KeyEvent
import android.view.LayoutInflater
import android.view.View
import android.view.ViewGroup
import androidx.fragment.app.Fragment
import com.facebook.react.ReactApplication
import com.facebook.react.ReactDelegate
import com.facebook.react.ReactHost
import com.facebook.react.ReactNativeHost
import com.facebook.react.internal.featureflags.ReactNativeNewArchitectureFeatureFlags
import com.facebook.react.modules.core.PermissionAwareActivity
import com.facebook.react.modules.core.PermissionListener

/**
 * Fragment for creating a React View. This allows the developer to "embed" a React Application
 * inside native components such as a Drawer, ViewPager, etc.
 */
public open class ReactNativeFragment :
    Fragment(),
    PermissionAwareActivity {
    protected lateinit var reactDelegate: ReactDelegate
    private var disableHostLifecycleEvents = false
    private var permissionListener: PermissionListener? = null

    public override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        var mainComponentName: String? = null
        var launchOptions: Bundle? = null
        var fabricEnabled = false
        arguments?.let { args ->
            mainComponentName = args.getString(ARG_COMPONENT_NAME)
            launchOptions = args.getBundle(ARG_LAUNCH_OPTIONS)
            fabricEnabled = args.getBoolean(ARG_FABRIC_ENABLED)
            @Suppress("DEPRECATION")
            disableHostLifecycleEvents = args.getBoolean(ARG_DISABLE_HOST_LIFECYCLE_EVENTS)
        }
        checkNotNull(mainComponentName) { "Cannot loadApp if component name is null" }

        reactDelegate =
            if (ReactNativeNewArchitectureFeatureFlags.enableBridgelessArchitecture()) {
                ReactDelegate(requireActivity(), reactHost, mainComponentName, launchOptions)
            } else {
                @Suppress("DEPRECATION")
                (
                    ReactDelegate(
                    requireActivity(),
                    reactNativeHost,
                    mainComponentName,
                    launchOptions,
                    fabricEnabled,
                )
                )
            }
    }

    /**
     * Get the [com.facebook.react.ReactNativeHost] used by this app. By default, assumes [android.app.Activity.getApplication] is an
     * instance of [com.facebook.react.ReactApplication] and calls [com.facebook.react.ReactApplication.reactNativeHost]. Override this
     * method if your application class does not implement `ReactApplication` or you simply have a
     * different mechanism for storing a `ReactNativeHost`, e.g. as a static field somewhere.
     */
    @Suppress("DEPRECATION")
    @Deprecated(
        "You should not use ReactNativeHost directly in the New Architecture. Use ReactHost instead.",
        ReplaceWith("reactHost"),
    )
    protected open val reactNativeHost: ReactNativeHost?
        get() = (activity?.application as ReactApplication?)?.reactNativeHost

    /**
     * Get the [com.facebook.react.ReactHost] used by this app. By default, assumes [android.app.Activity.getApplication] is an
     * instance of [ReactApplication] and calls [ReactApplication.reactHost]. Override this method if
     * your application class does not implement `ReactApplication` or you simply have a different
     * mechanism for storing a `ReactHost`, e.g. as a static field somewhere.
     *
     * If you're using Old Architecture/Bridge Mode, this method should return null as [com.facebook.react.ReactHost] is
     * a Bridgeless-only concept.
     */
    protected open val reactHost: ReactHost?
        get() = (activity?.application as ReactApplication?)?.reactHost

    public override fun onCreateView(
        inflater: LayoutInflater,
        container: ViewGroup?,
        savedInstanceState: Bundle?,
    ): View? {
        reactDelegate.loadApp()
        return reactDelegate.reactRootView
    }

    public override fun onResume() {
        super.onResume()
        if (!disableHostLifecycleEvents) {
            reactDelegate.onHostResume()
        }
    }

    // This lines cause the trouble:

    // public override fun onPause() {
    //     super.onPause()
    //     if (!disableHostLifecycleEvents) {
    //         reactDelegate.onHostPause()
    //     }
    // }

    // public override fun onDestroy() {
    //     super.onDestroy()
    //     if (!disableHostLifecycleEvents) {
    //        reactDelegate.onHostDestroy()
    //     } else {
    //        reactDelegate.unloadApp()
    //     }
    // }

    @Deprecated("Deprecated in Java")
    public override fun onActivityResult(
        requestCode: Int,
        resultCode: Int,
        data: Intent?,
    ) {
        @Suppress("DEPRECATION")
        super.onActivityResult(requestCode, resultCode, data)
        reactDelegate.onActivityResult(requestCode, resultCode, data, false)
    }

    /**
     * Helper to forward hardware back presses to our React Native Host.
     *
     * This must be called via a forward from your host Activity.
     */
    public open fun onBackPressed(): Boolean = reactDelegate.onBackPressed()

    /**
     * Helper to forward onKeyUp commands from our host Activity. This allows [ReactFragment] to
     * handle double tap reloads and dev menus.
     *
     * This must be called via a forward from your host Activity.
     *
     * @param keyCode keyCode
     * @param event event
     * @return true if we handled onKeyUp
     */
    public open fun onKeyUp(
        keyCode: Int,
        event: KeyEvent,
    ): Boolean = reactDelegate.shouldShowDevMenuOrReload(keyCode, event)

    @Deprecated("Deprecated in Java")
    public override fun onRequestPermissionsResult(
        requestCode: Int,
        permissions: Array<String>,
        grantResults: IntArray,
    ) {
        @Suppress("DEPRECATION")
        super.onRequestPermissionsResult(requestCode, permissions, grantResults)
        permissionListener?.let {
            if (it.onRequestPermissionsResult(requestCode, permissions, grantResults)) {
                permissionListener = null
            }
        }
    }

    override fun checkPermission(
        permission: String,
        pid: Int,
        uid: Int,
    ): Int = getActivity()?.checkPermission(permission, pid, uid) ?: 0

    override fun checkSelfPermission(permission: String): Int = getActivity()?.checkSelfPermission(permission) ?: 0

    @Suppress("DEPRECATION")
    override fun requestPermissions(
        permissions: Array<String>,
        requestCode: Int,
        listener: PermissionListener?,
    ) {
        permissionListener = listener
        requestPermissions(permissions, requestCode)
    }

    /** Builder class to help instantiate a ReactFragment. */
    public class Builder {
        public var componentName: String? = null
        public var launchOptions: Bundle? = null
        public var fabricEnabled: Boolean = false

        /**
         * Set the Component name for our React Native instance.
         *
         * @param componentName The name of the component
         * @return Builder
         */
        public fun setComponentName(componentName: String): Builder {
            this.componentName = componentName
            return this
        }

        /**
         * Set the Launch Options for our React Native instance.
         *
         * @param launchOptions launchOptions
         * @return Builder
         */
        public fun setLaunchOptions(launchOptions: Bundle): Builder {
            this.launchOptions = launchOptions
            return this
        }

        public fun build(): ReactNativeFragment = newInstance(componentName, launchOptions, fabricEnabled)

        @Deprecated(
            "You should not change call ReactFragment.setFabricEnabled. Instead enable the NewArchitecture for the whole application with newArchEnabled=true in your gradle.properties file",
        )
        public fun setFabricEnabled(fabricEnabled: Boolean): Builder {
            this.fabricEnabled = fabricEnabled
            return this
        }
    }

    public companion object {
        protected const val ARG_COMPONENT_NAME: String = "arg_component_name"
        protected const val ARG_LAUNCH_OPTIONS: String = "arg_launch_options"
        protected const val ARG_FABRIC_ENABLED: String = "arg_fabric_enabled"

        @Deprecated("We will remove this and use a different solution for handling Fragment lifecycle events.")
        protected const val ARG_DISABLE_HOST_LIFECYCLE_EVENTS: String =
            "arg_disable_host_lifecycle_events"

        /**
         * @param componentName The name of the react native component
         * @param launchOptions The launch options for the react native component
         * @param fabricEnabled Flag to enable Fabric for ReactFragment
         * @return A new instance of fragment ReactFragment.
         */
        private fun newInstance(
            componentName: String?,
            launchOptions: Bundle?,
            fabricEnabled: Boolean,
        ): ReactNativeFragment {
            val args =
                Bundle().apply {
                    putString(ARG_COMPONENT_NAME, componentName)
                    putBundle(ARG_LAUNCH_OPTIONS, launchOptions)
                    putBoolean(ARG_FABRIC_ENABLED, fabricEnabled)
                }
            return ReactNativeFragment().apply { setArguments(args) }
        }
    }
}

```

This prevents React Native from stopping execution. I’m sure there are better solutions, since the JS code never unloads now, but it works perfectly fine in our case, so we can leave it as is.

## iOS

Again, we followed the [documentation](https://docs.expo.dev/brownfield/get-started/#create-the-reactviewcontroller) exactly.

Now, the user can open the first screen and navigate to nested RN screens without problems. However, something glitchy happens if the user goes back to the first screen and then tries to open another RN screen inside React Navigation. The app crashes with a mysterious error:

```
Exception	NSException *	"-[RCTView setOnDisappear:]: unrecognized selector sent to instance 0x15936ca80"	0x0000000145ea9530
```

It turns out you must have only one RN View Factory in your application. The fix is simple: use a single static instance instead of creating a new one for every screen:

```swift
final class ReactNativeVC: UIViewController {
    public static var reactNativeFactory: RCTReactNativeFactory?
    public static var reactNativeDelegate: RCTReactNativeFactoryDelegate?
    private var componentName: String
    private var props: [String: Any]?

    override func viewDidLoad() {
        super.viewDidLoad()

        let rootViewProps: [String: Any] = []

        if (ReactNativeVC.reactNativeDelegate == nil || ReactNativeVC.reactNativeFactory == nil) {
            ReactNativeVC.reactNativeDelegate = ReactNativeDelegate()
            ReactNativeVC.reactNativeDelegate!.dependencyProvider = RCTAppDependencyProvider()
            ReactNativeVC.reactNativeFactory = RCTReactNativeFactory(
                delegate: ReactNativeVC.reactNativeDelegate!
            )
        }
        view = ReactNativeVC.reactNativeFactory!.rootViewFactory.view(
            withModuleName: self.componentName,
            initialProperties: rootViewProps
        )
        ReactNativeVC.reactNativeDelegate!.setRootView(view, toRootViewController: self)
    }

    // ... same code as in documentation
}
```

Now everything works as expected on iOS as well. Users can freely navigate between RN screens using either React Navigation or native navigation, in any order and at any depth.

I hope this helps anyone struggling with the challenging task of React Native brownfield integration!
