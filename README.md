# three-fiber-native-gesture-tutorial

## Introduction

[React-three-fiber](https://github.com/pmndrs/react-three-fiber) is a React renderer for `Three.js`. It allows us to express `Three.js` in JSX, and there are also many helper functions available in [react-three/drei](https://github.com/pmndrs/drei). However, neither `react-three-fiber` nor `react-three/drei` offer an easy-to-use way for gesture interaction in React Native. In this tutorial (probably the first tutorial on the internet) you will learn how to implement gesture control, especially the gesture control for cameras, in `react-three-fiber` in React Native.

So, the core question is: if `react-three-fiber` and `react-three/drei` don't have the ability for gesture control in React Native, then how can we implement it on our own? The answer is quite tricky: we can have a `<View>` over the `<Canvas>` component. Then we monitor the gestures on the `<View>` instead of any `Three.js` components, and lastly, we handle the gestures by changing the status of those `Three.js` components in response.

## Use react-native-gesture-handler

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

## Use react-spring

As you might notice, in the official document of `react-native-gesture-handler` (as well as the quick start), they recommend using `react-native-reanimated` at the same time. However, it is not our case.  `React-native-reanimated` doesn't NOT work for `Three.js`. In `react-three-fiber`'s official document, the only animation package mentioned is (react-spring)[https://www.react-spring.dev/]. We are going to use it.

Installation command for `react-spring`:

```bash
npm install three @react-spring/three
```

A short tutorial about how to use `react-spring` in `react-three-fiber`: https://docs.pmnd.rs/react-three-fiber/tutorials/using-with-react-spring

## Build a gesture-controllable custom perspective camera

Cool! The next issue we need to deal with is how to combine these two packages with `react-three-fiber`. Let's build a gestured perspective camera as an example.

To start with, we need to have a custom camera to be able to animate the camera. The `animated.perspectiveCamera` component is from `@react-spring/three`.

```react
const Camera = (props) => {
  const cameraRef = useRef();
  const set = useThree(({ set }) => set);
  const size = useThree(({ size }) => size);
  useLayoutEffect(() => {
    if (cameraRef.current) {
      cameraRef.current.aspect = size.width / size.height;
      cameraRef.current.updateProjectionMatrix();
    }
  }, [size, props]);

  useLayoutEffect(() => {
    set({ camera: cameraRef.current });
  }, []);

  return (
    <animated.perspectiveCamera
      ref={cameraRef}
      fov={75}
      near={0.1}
      far={100}
      position={props.position}
    />
  );
};
```

Now, if we use this custom in `<Canvas>`, it will replace the default one.

```react
<Canvas>
    <Camera position={position} />
</Canvas>
```

Next, we need to bind the gesture handlers with the animation on the custom camera. The idea is straightforward: bind the pan gesture to move the camera in the XY plane, and pinch gesture for zoom in and out (changing position Z).

Before having the gestures, we need to have our animation springs first.

```react
const [{ position }, api] = useSpring(() => ({
    from: { position: [0, 0, 5] },
    to: { position: [0, 0, 10] },
}));
```

The starting position of the camera is (0, 0, 10). Then let's set the pan gesture. When users are panning, the event can monitor the position translation information as `translationX` and `translationY`. What we need to do is just call the spring api to set a new position in terms of the original position (`savedPos`) and the translation information we knew. We need to set the `immediate` parameter to true so that the animation will be shown immediately.

```react
let savedPos = [0, 0, 10];
const panGesture = Gesture.Pan()
.onUpdate((e) => {
    // console.log(e.translationX, e.translationY);
    api.start({
        from: { position: savedPos },
        to: {
            position: [
                savedPos[0] - e.translationX / 100,
                savedPos[1] + e.translationY / 100,
                savedPos[2],
            ],
        },
        immediate: true,
    });
})
.onEnd(() => {
    savedPos = position.goal;
});
```

As for the pinch gesture, the `scale` is monitored. When scale is between 0 and 1 we should zoom out, and when scale is greater than 1 we should zoom in.

```react
const pinchGesture = Gesture.Pinch()
.onUpdate((e) => {
    console.log(e.scale);
    api.start({
        from: { position: savedPos },
        to: {
            position: [
                savedPos[0],
                savedPos[1],
                savedPos[2] - (e.scale - 1) * 5,
            ],
        },
        immediate: true,
    });
})
.onEnd(() => {
    savedPos = position.goal;
});
```

Combine the two gestures together.

```react
const composed = Gesture.Race(panGesture, pinchGesture);
```

Lastly, we can pass the `composed` gesture in our `<GestureDetector>`.

```react
return (
    <GestureDetector gesture={composed}>
        <View style={styles.canvasContainer}>
            <Canvas>
                <Camera position={position} />
            </Canvas>
        </View>
    </GestureDetector>
);
```

The complete code:

```react
import React, { useRef, useState, useLayoutEffect } from "react";
import { StyleSheet, View } from "react-native";
import { Canvas, useFrame, useThree } from "@react-three/fiber/native";
import {
  GestureDetector,
  Gesture,
} from "react-native-gesture-handler";
import { animated, useSpring } from "@react-spring/three";

function Box(props) {
  const mesh = useRef(null);
  const [hovered, setHover] = useState(false);
  const [active, setActive] = useState(false);
  useFrame((state, delta) => (mesh.current.rotation.x += 0.01));
  return (
    <mesh
      {...props}
      ref={mesh}
      scale={active ? 1.5 : 1}
      onClick={(event) => setActive(!active)}
      onPointerOver={(event) => setHover(true)}
      onPointerOut={(event) => setHover(false)}
    >
      <boxGeometry args={[1, 1, 1]} />
      <meshStandardMaterial color={hovered ? "hotpink" : "orange"} />
    </mesh>
  );
}

const Camera = (props) => {
  const cameraRef = useRef();
  const set = useThree(({ set }) => set);
  const size = useThree(({ size }) => size);
  useLayoutEffect(() => {
    if (cameraRef.current) {
      cameraRef.current.aspect = size.width / size.height;
      cameraRef.current.updateProjectionMatrix();
    }
  }, [size, props]);

  useLayoutEffect(() => {
    set({ camera: cameraRef.current });
  }, []);

  return (
    <animated.perspectiveCamera
      ref={cameraRef}
      fov={75}
      near={0.1}
      far={100}
      position={props.position}
    />
  );
};

export default function App() {
  const [{ position }, api] = useSpring(() => ({
    from: { position: [0, 0, 5] },
    to: { position: [0, 0, 10] },
  }));
  let savedPos = [0, 0, 10];
  const panGesture = Gesture.Pan()
    .onUpdate((e) => {
      // console.log(e.translationX, e.translationY);
      api.start({
        from: { position: savedPos },
        to: {
          position: [
            savedPos[0] - e.translationX / 100,
            savedPos[1] + e.translationY / 100,
            savedPos[2],
          ],
        },
        immediate: true,
      });
    })
    .onEnd(() => {
      savedPos = position.goal;
    });

  const pinchGesture = Gesture.Pinch()
    .onUpdate((e) => {
      console.log(e.scale);
      api.start({
        from: { position: savedPos },
        to: {
          position: [
            savedPos[0],
            savedPos[1],
            savedPos[2] - (e.scale - 1) * 5,
          ],
        },
        immediate: true,
      });
    })
    .onEnd(() => {
      savedPos = position.goal;
    });

  const composed = Gesture.Race(panGesture, pinchGesture);

  return (
    <View style={styles.container}>
      <GestureDetector gesture={composed}>
        <View style={styles.canvasContainer}>
          <Canvas>
            <Camera position={position} />
            <ambientLight />
            <pointLight position={[10, 10, 10]} />
            <Box position={[-1.2, 0, 0]} />
            <Box position={[1.2, 0, 0]} />
          </Canvas>
        </View>
      </GestureDetector>
    </View>
  );
}

const styles = StyleSheet.create({
  container: {
    flex: 1,
    backgroundColor: "#000",
    alignItems: "center",
    justifyContent: "center",
  },
  canvasContainer: {
    width: "95%",
    height: "85%",
    backgroundColor: "#fff",
  },
});
```

The final work is not perfect, because we didn't offer a way to rotate the camera or change the camera's facing direction. However, it is very simple to implement it - just pick another gesture and follow the same way. Try it yourself!

## Demo Project

Here is the link of the demo project:

https://github.com/migodz/three-fiber-native-gesture-demo