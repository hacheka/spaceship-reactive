# SPACESHIP REACTIVE!

My implementation of the game 'Spaceship Reactive' of the book ['Reactive Programming with RxJS'](https://pragprog.com/book/smreactjs/reactive-programming-with-rxjs)

### Modifications

#### Hero shots

The code that appears in the book renders the whole scene using the operator _combineLatest_. This operator waits to emit the first element until every input observable has emitted at least one value.

One the observables is _heroShots_, that depend on _playerFiring_ that emits an element (fires a shot) when the mouse is clicked or the spacebar pressed.

```
var playerFiring = Rx.Observable
  .merge(
    Rx.Observable.fromEvent(canvas, 'click'),
    Rx.Observable.fromEvent(canvas, 'keydown')
      .filter(function (evt) { return evt.keycode === 32; })
  )
```

What happens then? The scene waits to be rendered after the player fires because of _combineLatest_.

The obvious solution is to emit a first value using _startWith_. The problem is that with this approach one shot will be automatically triggered when the game starts. So we must make another modification to our observables.

My proposed solution is to use a index number for the shots instead of a timestamp and then use that index to avoid using the first shot (neither painting it nor checking its collisions).

For creating a index for every shot we can use _scan_ operator.

```
var playerFiring = Rx.Observable
  .merge(
    Rx.Observable.fromEvent(canvas, 'click'),
    Rx.Observable.fromEvent(canvas, 'keydown')
      .filter(function (evt) { return evt.keycode === 32; })
  )
  .sample(200)
  .startWith(0)
  .scan(function (acc, current) {
    return ++acc;
  }, -1);
 ```
 
 Then, in the _heroShots_ observable we use that instead instead of the timestamp used in the book.
 
 ```
var heroShots = Rx.Observable
  .combineLatest(
    playerFiring,
    spaceship,
    function (shotNumber, spaceship) {
      return {
        index: shotNumber,
        x: spaceship.x
      };
    }
  )
  .distinctUntilChanged(function (shot) { return shot.index; })
  .scan(function (shotArray, shot) {
    shotArray.push({x: shot.x, y: HERO_Y, index: shot.index });
    return shotArray;
  }, []);
 ```
 
 As the last step, we skip the very first shot to be painted.
 
 ```
 function paintHeroShots(heroShots, enemies) {
  heroShots.forEach(function (shot, i) {
    if (shot.index > 0) {
    (...)
```
 
 Et voila! The game will start as expected, without waiting on the player to click anything.

