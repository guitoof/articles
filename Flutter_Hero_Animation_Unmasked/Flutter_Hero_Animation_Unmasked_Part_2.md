# Flutter Hero Animation Unmasked - Part 2/2

Welcome back to "Hero Animation Unmasked", our little adventure to recode the flutter Shared Element Transition called "Hero Animation".

## Previously in Hero Animation Unmasked

In the previous part: [Hero Animation Unmasked - Part 1](https://blog.bam.tech/en/developer-news/flutter-hero-animation-unmasked), we have ...

1. understood the general mechanism of the Hero Animation
2. created our own `UnmaskedHero` widget and its `UnmaskedHeroController`
3. hooked ourselves to the `didPush` Navigation method to react to navigation
4. browsed the Elements Tree to look for interesting Widget instances
5. displayed our `UnmaskedHero` into the screen overlay

Now that we've managed to find our Hero-wrapped widgets and display them onto the screen...

<div style="text-align: center;">
  <img src="https://media.giphy.com/media/eIUpSyzwGp0YhAMTKr/giphy.gif" height="200" style="border-radius: 20px;" alt="Display Hero on Overlay" />
</div>

## Let's make them Fly!

In order to so, we'll implement the following steps:

1. Display our `UnmaskedHeroes` at their initial position on screen
2. Animate them from their initial to their final positions
3. Hide the source & destination widgets during the flight

Eventually, we'll buy our `UnmaskedHero` a return ticket by making sure they can fly back when we navigate back

### 1. 🛫 Ready for Take-off?

<div style="text-align: center;">
  <img src="https://media0.giphy.com/media/gHhJI8R5tizPIa83hc/200w.gif?cid=82a1493brcxsoyswlo5loj8ntpvebr1g9ndkr1l3q8hpci9x&rid=200w.gif&ct=v" height="200" style="border-radius: 20px;" alt="Ready for Take-off" />
</div>

#### Compute Hero from/to locations on screen

Commit: [7f37e14b1d45335f9044fba6187d83ec3ccb0350](https://github.com/guitoof/hero-animation-unmasked/commit/7f37e14b1d45335f9044fba6187d83ec3ccb0350)

In order to make our Hero fly, we first need to compute the locations on the screen from and to which they should be flying.
To do so, in the `UnmaskedHeroController` class, we create a `_locateHero` method that, given the `UnmaskedHeroState` and a `BuildContext`, will return a `Rect` object holding the actual onscreen position of the associated element.

```dart
/// lib/packages/unmasked_hero/unmasked_hero_controller.dart

  ...
  /// Locate Hero from within a given context
  /// returns a [Rect] that will hold the hero's position & size in the context's frame of reference
  Rect _locateHero({
    required UnmaskedHeroState hero,
    required BuildContext context,
  }) {
    final heroRenderBox = (hero.context.findRenderObject() as RenderBox);
    final RenderBox ancestorRenderBox = context.findRenderObject() as RenderBox;
    assert(heroRenderBox.hasSize && heroRenderBox.size.isFinite);

    return MatrixUtils.transformRect(
      heroRenderBox.getTransformTo(ancestorRenderBox),
      Offset.zero & heroRenderBox.size,
    );
  }
```

Here, we access the `RenderObject` of the hero by calling `hero.context.findRenderObject()` and the `RenderObject` of the global context (which we refer to as its ancestor): `context.findRenderObject()`.
Then we compute the [**transformation matrix**](https://en.wikipedia.org/wiki/Transformation_matrix) that describes the geometric transformation between 2 `RenderObject` using the `getTransformTo` method.
Finally, we apply this transformation using `MatrixUtils.transformRect` to return our hero's location in the frame of reference of the given context.

In the `didPush` method, let's now call the `_locateHero` method for both source and destination to compute the from and to position:

```dart
for (UnmaskedHeroState hero in destinationHeroes.values) {
    final UnmaskedHeroState? sourceHero = sourceHeroes[hero.widget.tag];
    final UnmaskedHeroState destinationHero = hero;
    if (sourceHero == null) {
        print(
            'No source Hero could be found for destination Hero with tag: ${hero.widget.tag}');
        continue;
    }

    final Rect fromPosition =
        _locateHero(hero: sourceHero, context: fromContext);
    final Rect toPosition =
        _locateHero(hero: destinationHero, context: toContext);
    print('Hero will fly from $fromPosition to $toPosition');

    _displayFlyingHero(hero);
}
```

<div style="text-align: center;">
  <img src="https://media.giphy.com/media/l378afApZWr0IyIZq/giphy.gif" height="200" style="border-radius: 20px;" alt="Compute locations" />
</div>

#### Display flying Hero at source position

Commit: [ef3c500cd5271cbd30f785d541f2e78108a42040](https://github.com/guitoof/hero-animation-unmasked/commit/ef3c500cd5271cbd30f785d541f2e78108a42040)

Now that our Hero's initial and final positions are computed. We need to make them initially appear at the "from" position. For that, let's rename the `displayFlyingHero` method with a more explanatory name: `_displayFlyingHeroAtPosition` and pass it an additional `Rect` parameter that will hold the position at which we want to display the Hero:

```dart
/// lib/packages/unmasked_hero/unmasked_hero_controller.dart

/// Display Hero on Overlay at the given position
void _displayFlyingHeroAtPosition({
required UnmaskedHeroState hero,
required Rect position,
}) {

...

}

...

@override
void didPush(Route<dynamic> toRoute, Route<dynamic>? fromRoute) {
    ...

    final Rect fromPosition = _locateHero(hero: sourceHero, context: fromContext);
    final Rect toPosition = _locateHero(hero: destinationHero, context: toContext);
    _displayFlyingHeroAtPosition(hero: hero, position: fromPosition);
}
```

Next, we replace the `Container` returned by the `builder` of the `OverlayEntry` with a `Positioned` widget and pass it the given position:

```dart
/// lib/packages/unmasked_hero/unmasked_hero_controller.dart

overlayEntry = OverlayEntry(
    builder: (BuildContext context) => Positioned(
    child: hero.widget.child,
    top: position.top,
    left: position.left,
    width: position.width,
    height: position.height,
    ),
);
```

<div style="text-align: center;">
  <img src="./images/Unmasked_Hero_Initial_Position.gif" height="300" style="border-radius: 20px;" alt="Transition without Hero animation" />
</div>

The overlayed Hero now appears at the same position as the source Hero widget.

### 2. 🛩 Animate hero between from/to positions

<div style="text-align: center;">
  <img src="https://media.giphy.com/media/10bKPDUM5H7m7u/giphy.gif" height="140" style="border-radius: 20px;" alt="Compute locations" />
</div>

Commit: [daa20dd0837a87659885411c8de0c591643d7bd8](https://github.com/guitoof/hero-animation-unmasked/commit/daa20dd0837a87659885411c8de0c591643d7bd8)

Eventually, here comes the actual "Flying" part we've been waiting for 😅!
To do so, we'll create a widget responsible for the animation and display it on the overlay:

First, in the `unmasked_hero` folder, create a `FlyingUnmaskedHero` widget:

```dart
/// lib/packages/unmasked_hero/flying_unmasked_hero.dart

import 'dart:async';

import 'package:flutter/widgets.dart';

class FlyingUnmaskedHero extends StatefulWidget {
  final Rect fromPosition;
  final Rect toPosition;
  final Widget child;

  FlyingUnmaskedHero({
    required this.fromPosition,
    required this.toPosition,
    required this.child,
  });

  @override
  FlyingUnmaskedHeroState createState() => FlyingUnmaskedHeroState();
}

class FlyingUnmaskedHeroState extends State<FlyingUnmaskedHero> {
  bool flying = false;

  @override
  void initState() {
    Timer(Duration(milliseconds: 0), () {
      setState(() {
        flying = true;
      });
    });
    super.initState();
  }

  @override
  Widget build(BuildContext context) {
    final Rect fromPosition = widget.fromPosition;
    final Rect toPosition = widget.toPosition;
    return AnimatedPositioned(
      child: widget.child,
      duration: Duration(milliseconds: 200),
      top: flying ? toPosition.top : fromPosition.top,
      left: flying ? toPosition.left : fromPosition.left,
      height: flying ? toPosition.height : fromPosition.height,
      width: flying ? toPosition.width : fromPosition.width,
    );
  }
}
```

This widget is a simple `StatefulWidget` responsible for handling the animation between the initial and final position of our hero that we pass as parameters.

For the sake of simplicity, we use an `AnimatedPositioned` widget to handle the animation between the 2 positions. The actual Hero widget uses the lower-level API [`RectTween`](https://api.flutter.dev/flutter/animation/RectTween-class.html). This induces a couple of notable things here:

1. We hardcode the duration of the animation to `200`. In real life, we would want to ensure the animation duration matches the animation of the navigation between the 2 pages.
2. We use a `Timer` of `0 milliseconds` in the `initState` method to ensure the widget is built once with the `flying` state set to false, which initialize the position to `toPosition` before being animated to toward the `fromPosition`.

Next, we rename the `_displayFlyingHeroAtPosition` to `_startFlying` and pass it both the from and to positions:

```dart
/// lib/packages/unmasked_hero/unmasked_hero_controller.dart


    /// Display Hero on Overlay and animate them from 1 position to the next
    void _startFlying({
        required UnmaskedHeroState hero,
        required Rect fromPosition,
        required Rect toPosition,
    }) {

    ...

    _startFlying(hero: hero, fromPosition: fromPosition, toPosition: toPosition);
```

Finally, we can replace the widget built by the `OverlayEntry` to use our `FlyingUnmaskedHero`:

```dart
overlayEntry = OverlayEntry(
    builder: (BuildContext context) => FlyingUnmaskedHero(
        fromPosition: fromPosition,
        toPosition: toPosition,
        child: hero.widget.child,
    ),
);
```

... is it a bird 🦅 ? is it a plane 🛩 ? 🤩

<div style="text-align: center;">
  <img src="./images/Flying_Unmasked_Hero_Raw.gif" height="300" style="border-radius: 20px;" alt="Flying Unmasked Hero" />
</div>

Our hero animates nicely between the initial and final positions 💪.

### 3. 🧹 Clean up

We're almost there. Now we just need to make sure that the original widgets and the one flying onto the overlay are not displayed simultaneously to produce the illusion that they are the same widget.
In order to produce this illusion, there are 2 things left to do:

1. Remove the widget from the overlay when the animation ends
2. Hide our `UnmaskedHero` widget's child during the animation

<div style="text-align: center;">
  <img src="https://media.giphy.com/media/SABGACIrfegQ4O1Aey/giphy.gif" height="200" style="border-radius: 20px;" alt="Compute locations" />
</div>

#### Remove the widget from the overlay when the animation ends

Commit: [dc3e4b6b222d6c0ad004b958464ae5e14466b714](https://github.com/guitoof/hero-animation-unmasked/commit/dc3e4b6b222d6c0ad004b958464ae5e14466b714)

Add a `onFlyingEnded` callback property to the `FlyingUnmaskedHero` widget to be able to listen to this event:

```dart
/// lib/packages/unmasked_hero/flying_unmasked_hero.dart

class FlyingUnmaskedHero extends StatefulWidget {
  final Rect fromPosition;
  final Rect toPosition;
  final Widget child;
  final VoidCallback? onFlyingEnded;  /// <--- Add this line

  FlyingUnmaskedHero({
    required this.fromPosition,
    required this.toPosition,
    required this.child,
    this.onFlyingEnded,   /// <--- Add this line
  });

    @override
    Widget build(BuildContext context) {
      ...

        return AnimatedPositioned(
            child: widget.child,
            duration: Duration(milliseconds: 200),
            top: flying ? toPosition.top : fromPosition.top,
            left: flying ? toPosition.left : fromPosition.left,
            height: flying ? toPosition.height : fromPosition.height,
            width: flying ? toPosition.width : fromPosition.width,
            onEnd: widget.onFlyingEnded,   /// <--- Add this line
        );
    }
}
```

and pass it a function that removes the overlayEntry:

```dart
/// lib/packages/unmasked_hero/unmasked_hero_controller.dart

...

    OverlayEntry? overlayEntry;
    overlayEntry = OverlayEntry(
      builder: (BuildContext context) => FlyingUnmaskedHero(
          fromPosition: fromPosition,
          toPosition: toPosition,
          child: hero.widget.child,
          onFlyingEnded: () {           /// <--- Add
            overlayEntry?.remove();     /// <--- these
          }),                           /// <--- lines
    );
```

#### Hide hero's child during animation

Commit: [7836d20d8fa1ddba160aab782aaf689c92899b3e](https://github.com/guitoof/hero-animation-unmasked/commit/7836d20d8fa1ddba160aab782aaf689c92899b3e)

We first modify our `UnmaskedHeroState` to add the capibility to hide or show its child:

```dart
/// lib/packages/unmasked_hero/unmasked_hero.dart

class UnmaskedHeroState extends State<UnmaskedHero> {
  bool hidden = false;

  void hide() {
    setState(() {
      hidden = true;
    });
  }

  void show() {
    setState(() {
      hidden = false;
    });
  }

  @override
  Widget build(BuildContext context) {
    return Opacity(opacity: hidden ? 0.0 : 1.0, child: widget.child);
  }
}
```

We now modify the `startFlying` method to pass both `sourceHero` and `destinationHero`

```dart
/// lib/packages/unmasked_hero/unmasked_hero_controller.dart

void _startFlying({
    required UnmaskedHeroState sourceHero,      /// <--- Replace hero by sourceHero here
    required UnmaskedHeroState destinationHero, /// <--- Add this line
    required Rect fromPosition,
    required Rect toPosition,
}) {
    ...


    _startFlying(
        sourceHero: sourceHero,             /// <--- Replace hero by sourceHero here
        destinationHero: destinationHero,   /// <--- Add this line
        fromPosition: fromPosition,
        toPosition: toPosition);
    }
```

And finally, we add the logic:

1. Hide the widgets before the animation starts
2. Show the widgets once it ended

```dart
/// lib/packages/unmasked_hero/unmasked_hero_controller.dart

    ...

    /// Hide source & destination heroes during flight animation
    sourceHero.hide();
    destinationHero.hide();

    OverlayEntry? overlayEntry;
    overlayEntry = OverlayEntry(
      builder: (BuildContext context) => FlyingUnmaskedHero(
          fromPosition: fromPosition,
          toPosition: toPosition,
          child: hero.widget.child,
          onFlyingEnded: () {
            /// Show source & destination heroes at the end of flight animation
            sourceHero.show();
            destinationHero.show();
            overlayEntry?.remove();
          }),
    );
```

<div style="text-align: center;">
  <img src="./images/Unmasked_Hero_final_forward_only.gif" height="300" style="border-radius: 20px;" alt="Unmasked Hero Animation (forward)" />
</div>

Well.. this starts to look very much like a proper Shared Element Transition doesn't it?

### 4. ⏮ Be kind, rewind

Alright, I heard you... it does not quite because the animation is not performed backward when we navigate back to the source page 😏.

<div style="text-align: center;">
  <img src="https://theamazingjeanette.files.wordpress.com/2016/08/tumblr_mdezrnan3j1rvcg92o1_500.gif?w=371" height="170" style="border-radius: 20px;" alt="Compute locations" />
</div>

Let's take this last step together:

#### Refactor hero flying logic to use didPop

Commit: [add9e0e1fcbf282f1ce817d8eb86f34a0c64311c](https://github.com/guitoof/hero-animation-unmasked/commit/add9e0e1fcbf282f1ce817d8eb86f34a0c64311c)

In the `UnmaskedHeroController`, proceed to an "extract method" type of refactoring to move the entire content of the `didPush` method inside a `_flyFromTo` method:

```dart
    void _flyFromTo(        /// <--- Replace didPush by _flyFromTo
        Route<dynamic>? fromRoute,
        Route<dynamic> toRoute,
    ) {

...

    @override
    void didPush(Route<dynamic> toRoute, Route<dynamic>? fromRoute) {
        _flyFromTo(fromRoute, toRoute);     /// <---- Call it here
        super.didPush(toRoute, fromRoute);
    }
```

and eventually use it inside the overriden `didPop` method of the controller:

```dart
    @override
    void didPop(Route<dynamic> fromRoute, Route<dynamic>? toRoute) {
        if (toRoute == null) {
        return;
        }
        _flyFromTo(fromRoute, toRoute);
        super.didPop(fromRoute, toRoute);
    }
```

And there you go...

<div style="text-align: center;">
  <img src="./images/Unmasked_Hero_Animation.gif" height="300" style="border-radius: 20px;" alt="Unmasked Hero Animation (forward)" />
</div>

Our `UnmaskedHero` smoothly flies back and forth from one page to the other, just as the original [`Hero` widget](https://flutter.dev/docs/development/ui/animations/hero-animations).

### 🌯 Let's wrap it up

Our great adventure of recoding the Flutter Hero animation comes to an end. Let's take a look back at our journey:

In the 1st part of this article: [Flutter Hero Animation Unmasked - Part 1/2](https://blog.bam.tech/en/developer-news/flutter-hero-animation-unmasked), we have:

1. understood the general mechanism of the Hero Animation
2. created our own `UnmaskedHero` widget and its `UnmaskedHeroController`
3. hooked ourselves to the `didPush` Navigation method to react to navigation
4. browsed the Elements Tree to look for interesting Widget instances
5. displayed our `UnmaskedHero` into the screen overlay

in this 2nd part, we have: 6. computed the initial and final position on screen of our widget 7. animated them onto the overlay from initial to final position 8. produced the illusion of the widget "moving" by hiding the original ones during the animation and removing the overlayed one afterwards 9. added support for the backward animation when navigating back from the destination page

I really hope that you enjoyed this journey to Unmasked the Hero Animation as much as I did and that you learned some things.
The Hero widget has no secrets for you now 😉.
If you want to dive even deeper in your understanding of this animation, check out the [source code of the actual Hero widget](https://github.com/flutter/flutter/blob/master/packages/flutter/lib/src/widgets/heroes.dart)

If you have any questions, if you find any mistakes, inaccuracies, or if you just want to have a chat about Flutter, I'd be very happy to. You can find me on Twitter [@Guitoof](https://twitter.com/guitoof) or on the [Flutter Community Slack](https://fluttercommunity.slack.com/) where I also go by the name `Guitoof`.
