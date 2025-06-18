# Traffic Simulation

## Code Structure

Throughout the code, the values will be in SI units. Based on a scaling factor, that will be converted to pixels on the screen. Likewise, throughout animation, the movement of an object will be calculated using the frame rate. So a vector in units of m/s will be converted to pixels/frame. 

### Create scene

The script starts by setting up the scene based on certain parameters. Global parameters, such as lanes will have default values that user can change when setting up a scenario. 

Parameters:
```js
const lanes = [
  {
    start: 0,
    end: 50
  }
  {
    start: 0,
    end: 100
  }
  {
    start: 0,
    end: 100
  }
]
const cars = [Car]
const max_speed = float
```

The lowest index, `lane[0]` in this case, will be the "slowest" lane, and the highest indexed lane will be the "fastest" lane. However, that shouldn't matter to the simulation because the cars will always be able to pass on either left or right of the car in front. The clarification is mainly that, for the cars perspective, `lane[0]` is the furthest right when driving forward. 

The scene will generate cars with varying private attributes and globally accessbile positions. 


### Simulate Scene

**App-level logic**

Render cars based on position, size, and signal. Handle instantiating cars at random intervals at beginning of lanes. Delete cars after they are outside of boundary. 

```js
getCar(id) {
  return cars.filter(item => item.id == id);
}
getCars(ids) {
  return ids.forEach(item => getCar(id));
}
instantiateCar() {
  ...//update cars list with new Car object with psuedo-random parameters
}
simulate() {
  ...//run at specified framerate, rendering each car based on pos. etc. 
}
```

**Car-level logic**

Car object is defined with the following attributes and methods.
```js
class Car {
  id: int,
  x: float > 0,
  lane: int range(0, maxlane),
  speed: float,
  acc_dir: bool, // gas 1, brake 0
  acc_mag: float, // constant

  forward: null || car_id,
  adjacent: null || [car_id],
  comfortable_distance: float range(5, 10),
  safety: float range(0,1),
  size: float = 4,
  response_time: float range(0.5, 1.5),
  signal_on: bool,
  signal_dir: bool, // right 1, left 0
  lane_change_count: float,
  speed_comfort: float,
  

  get_car_forward()
  get_cars_adjacent()
  update_position()
}
```

Here is the description of how the car will behave basedo on these parameters and its surroundings. 

- Every frame, find cars in front and adjacent
  - `forward` type object `Car` - look for car with lowest `x` > `this.x`
  - `adjacent` type object array `[Car]` - look for car in `this.lane` $\pm 1$ that has `x` in `this.x` $\pm$ `abs(this.size/2 + this.comfortable_distance)`
- Every frame, adjust position
  - Using current speed and acceleration:
    - Add `this.speed/framerate` to `this.x`.
    - Add or subtract `this.acc_mag/framerate` to `this.speed`
    - Always `this.speed >= 0`
  - Handle what `acc_dir` (brake or gas) is appropriate:
    - First, slow down if speeding. 

      ```js 
      if (this.speed > this.speed_comfort + max_speed) this.acc_dir = 0; 
      ```

    - If not, check if need to slow down to avoid crashing. 
      
      ```js
      else if (getCar(this.forward).x < this.x + this.comfortable_distance) this.acc_dir = 0
      ```

    - If there is a car trying to lane shift into this lane, weighted on car's `saftey` attr, slow down, or ignore

      ```js
      // for each car in this.adjacent
      const shifting_into_lane (adj : Car) => { 
        // If it is in front of this car
        adj.x > this.x 
        // and has turn signal on
        && adj.signal_on 
        // and is shifting into the same lane as this car
        && (adj.signal_dir 
        && adj.lane == this.lane - 1 
        || !adj.signal_dir 
        && adj.lane == this.lane + 1);
      }
      // safety_attr will return either 0 or 1, more likely to be 1 if a safe driver
      const safety_attr = round.ceil(random()+this.safety)

      // This is the actual if statement
      else if (shifting_into_lane && safety_attr) this.acc_dir = 0
      ```

    - If not, and its clear ahead, speed up. 
      
      ```js
      else if (getCar(this.forward).x < this.x + this.comfortable_distance || !this.forward) this.acc_dir = 1
      ```
    
    Thus the car would naturally accelerate toward `comfortable_distance` if there is a car in front, or would accelerate toward `this.speed_comfort + max_speed` if there are no cars around.

- Lane changing logic: 
  - Compare current speed to `max_speed + this.speed_comfort`. 
    - If it is lower, and there is no adjacent car (`this.adjacent = []`), then the driver will want to turn. Turn `this.signal` to 1 and set direction to the direction with no adjacent cars (or random if no difference)

      ```js
      if (this.speed < max_speed + this.speed_comfort) {
        // check where adjacent cars are
        let right = false;
        let left = false;
        for (car of getCars(this.adjacent)) {
          if (car.lane == this.lane - 1) {right = true;}
          else if (car.lane == this.lane + 1) {left = true;}
        }
        // base direction off of adjacent cars location and turn on the signal
        this.signal_dir = right ? (left ? random : 0) : 1;
        this.signal_on = true;
      ```
  - If the signal is already on, increase `lane_change_count` every frame (in seconds) linearly from 0 to 3*`this.safety`, making the longest time anyone could change lanes 3 seconds.
  - If the signal is already on and `lane_change_count` reaches maximum, turn off signal and set lane. `this.lane = this.signal_dir ? this.lane + 1 : this.lane - 1`
  -  Set `lane_change_count` back to 0 when there is an adjacent car in lane in question. 
