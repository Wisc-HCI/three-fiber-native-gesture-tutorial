# three-fiber-native-gesture-tutorial

[React-three-fiber](https://github.com/pmndrs/react-three-fiber) is a React renderer for `Three.js`. It allows us to express `Three.js` in JSX, and there are also many helper functions available in [react-three/drei](https://github.com/pmndrs/drei). However, the neither `react-three-fiber` nor `react-three/drei` offer an easy-to-use way for gesture interaction in React Native. In this tutorial (probably the first tutorial on the internet) you will learn how to implement gesture control, especially the gesture control for cameras, in `react-three-fiber` in React Native.

So, the core question is: if `react-three-fiber` and `react-three/drei` don't have the ability for gesture control in React Native, then how can we implement it on our own? The answer is quite tricky: we can have a `<View>` over the `<Canvas>` component. Then we monitor the gestures on the `<View>` instead of any `Three.js` components, and lastly, we handle the gestures by changing the status of those `Three.js` components in response.

To achieve our goal, we will first need to use (react-native-gesture-handler)[https://docs.swmansion.com/react-native-gesture-handler/] which is a very popular touch system in React Native.

Installation command for `react-native-gesture-handler`:

```bash
npm install --save react-native-gesture-handler
```

Here is a quick start about how to use `react-native-gesture-handler`: https://docs.swmansion.com/react-native-gesture-handler/docs/quickstart/

To summarize, wrap your `<Canvas>` within the `<GestureDetector>`, and simply put your gesture handler as the parameter to the `<GestureDetector>`. Here is an example:

```react
export default function App() {
  const panGesture = Gesture.Pan()
    .onUpdate((e) => {
        // handle pan gesture when panning
    })
    .onEnd(() => {
        // handle pan gesture when gesture is done
    });

  const pinchGesture = Gesture.Pinch()
    .onUpdate((e) => {
        // handle pan gesture when pinching
    })
    .onEnd(() => {
        // handle pan gesture when gesture is done
    });

  // combine gestures together
  const composed = Gesture.Race(panGesture, pinchGesture);

  return (
    <View style={styles.container}>
      {/* put GestureDetector outside of Canvas with gesture handler */}
      <GestureDetector gesture={composed}>
          <Canvas>
			{/* your three.js componets */}
          </Canvas>
      </GestureDetector>
    </View>
  );
}
```

You might wanna ask: What if we use other gesture handling package? Will that work? Well, you can try it, but please confirm that the package is built for React Native instead of React Web before you use it. For example, the (@use-gesture)[https://use-gesture.netlify.app/] is a package for React Web and it will not work in mobile apps. It's a very bewildering name, right?

As you might notice, in the official document of `react-native-gesture-handler` (as well as the quick start), they recommend using `react-native-reanimated` at the same time. However, it is not our case.  `React-native-reanimated` doesn't NOT work for `Three.js`. In `react-three-fiber`'s official document, the only animation package mention is (react-spring)[https://www.react-spring.dev/]. We are going to use it.

Installation command for `react-spring`:

```bash
npm install three @react-spring/three
```

A short tutorial about how to use `react-spring` in `react-three-fiber`: https://docs.pmnd.rs/react-three-fiber/tutorials/using-with-react-spring



