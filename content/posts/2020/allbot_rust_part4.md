---
title: Learning embedded Rust by building RISC-V-powered robot - Part 4
date: 2020-04-19T00:00:00
categories: [projects, allbot-rust]
tags: [rust, riscv, allbot]
language: en
slug: learning-embedded-rust-by-building-riscv-powered-robot-part-4
---

Now that [we developed]({{< relref "/posts/2020/allbot_rust_part3" >}}) a high-level `Servo` trait that allows controlling servo motors, we can start writing code that will animate our spider bot by rotating multiple motors synchronously. The idea is that the robot's movement takes some fixed amount of time during which each moving motor may travel a different angle. Importantly, all moving motors should start and stop at the same time, and therefore, they will have different angular speeds. Such movements then are sequenced one after another to appear like a well-coordinated action.

To implement this in Rust, we introduce the `Move` structure that captures a reference to a servo motor and the desired degrees where it should be at the end of the animation.

```rust
struct Move<'a> {
    servo: &'a dyn Servo,
    desired: Degrees,
}

impl<'a> Move<'a> {
    fn new(servo: &'a dyn Servo, desired: Degrees) -> Move<'a> {
        Move { servo, desired }
    }
}

fn animate(moves: &[Move], speed: u32, sleep: &mut Sleep) -> () {
    const STEP_SPEED: u32 = 20;
    let step_num = speed / STEP_SPEED;

    let mut deltas: Vec<_, U8> = Vec::new();

    for m in moves {
        let current = m.servo.read();
        let delta = (m.desired.0 - current.0) / step_num as f64;

        deltas.push(delta).unwrap();
    }
    for _ in 0..(speed / STEP_SPEED) {
        for i in 0..moves.len() {
            let m = &moves[i];
            let delta = deltas[i];

            let new_degrees = m.servo.read().0 + delta;
            m.servo.write(new_degrees.degrees());
        }

        sleep.delay_ms(STEP_SPEED);
    }
}
```

The `animate` function takes a slice of references to `Move` structures, and the total time of animation (`speed`). Given that we have 20 milliseconds PWM cycle, it makes sense to change the servo angle at a step that is no shorter than 20 milliseconds. With fixed 20 milliseconds per step (`STEP_SPEED`), the `annotation` function computes the delta angle to move at every animation step. With this, it writes the angle for each servo motor in lockstep with sleeping in-between.

Note how we are using statically allocated `Vec` from an awesome [heapless](https://docs.rs/heapless) library, knowing that we have at most 8 moves that can participate in one animation. This invariant is enforced by unwrapping the `push` function result and therefore, will panic if slice with more than 8 moves was passed.

Also, note that we changed the `degrees` function to round up and down to the boundaries instead of panicking. In other words, it sets to 0 when the input is <0 and to 180 when it is >180. This way, we are safe with rounding errors.

Let's mount hip motors and see how this works in practice. The `ALLBOT` structure contains references to servo motors named by their placement in the robot's body. The `init` function sets all motors to 90 degrees. The `test` function animates the motors between 90 and 45 degrees with a given speed in milliseconds.

```rust
struct ALLBOT<'a> {
    hip_front_left: &'a dyn Servo,
    hip_front_right: &'a dyn Servo,
    hip_rear_left: &'a dyn Servo,
    hip_rear_right: &'a dyn Servo,
}

impl<'a> ALLBOT<'a> {
    fn init(&self) {
        self.hip_front_left.write(90.0.degrees());
        self.hip_front_right.write(90.0.degrees());
        self.hip_rear_left.write(90.0.degrees());
        self.hip_rear_right.write(90.0.degrees());
    }

    fn test(&self, sleep: &mut Sleep, speed: u32) {
        animate(
            &[
                Move::new(self.hip_front_left, 45.0.degrees(), speed),
                Move::new(self.hip_front_right, 45.0.degrees(), speed),
                Move::new(self.hip_rear_left, 45.0.degrees(), speed),
                Move::new(self.hip_rear_right, 45.0.degrees(), speed),
            ],
            speed,
            sleep,
        );

        animate(
            &[
                Move::new(self.hip_front_left, 90.0.degrees(), speed),
                Move::new(self.hip_front_right, 90.0.degrees(), speed),
                Move::new(self.hip_rear_left, 90.0.degrees(), speed),
                Move::new(self.hip_rear_right, 90.0.degrees(), speed),
            ],
            speed,
            sleep,
        );
    }
}
```

In the `main` function, we initialize `ALLBOT` by connecting specific servo motors in its body to the corresponding GPIO pins.

Note that we introduced the `inverted` flag as a last argument in the `PwmServo::new` that allows simplifying the animation code by making motors mirroring angles on opposite sides of the robot.

Finally, we call `test` animation in the loop with a speed of 2 seconds.

```rust
let mut sleep = Sleep::new(core_peripherals.clint.mtimecmp, clocks);

let allbot = ALLBOT {
    hip_front_left: &PwmServo::new(
        pin!(pins, dig4),
        &pwm1,
        1000,
        8400,
        false,
    ),
    hip_front_right: &PwmServo::new(
        pin!(pins, dig3),
        &pwm1,
        1000,
        8400,
        true,
    ),
    hip_rear_left: &PwmServo::new(
        pin!(pins, dig16),
        &pwm2,
        1000,
        8400,
        true,
    ),
    hip_rear_right: &PwmServo::new(
        pin!(pins, dig17),
        &pwm2,
        1000,
        8400,
        false,
    ),
};

allbot.init();

loop {
    allbot.test(&mut sleep, 2000);
}
```

Here is how the hip test looks with mounted upper legs.

![Hip test](/media/2020/allbot_rust_part4/hip_test.gif)

And the hip-and-knee test with all 8 servo motors and fully assembled legs.

![Hip and knee test](/media/2020/allbot_rust_part4/hip_and_knee_test.gif)

Now, we can copy the standard movements from the [original Arduino code](https://github.com/Velleman/ALLBOT-lib/blob/master/examples/VR408/VR408.ino). Let's make our robot cute and friendly by using the sequence of `look_right`, `wave_front_right`, `look_left`, and `wave_front_left`!

![Hello](/media/2020/allbot_rust_part4/hello.gif)

[Here](https://github.com/petrohi/allbot/blob/f0f6974f6c12d6d6221d123c2b452b75cddc42f2/src/main.rs) and [here](https://github.com/petrohi/allbot/blob/f672bee89cd340bf9fd4dcd5042c80d5256e33e7/src/main.rs) is the source code correspondingly for hip and hip-and-knee tests. The [final source](https://github.com/petrohi/allbot/blob/7f2b5a70f8636d417844ac7944c076bf6d8755d7/src/main.rs) code uses another excellent library--[wyhash](https://docs.rs/wyhash)--to select movements from the entire repertoire randomly.