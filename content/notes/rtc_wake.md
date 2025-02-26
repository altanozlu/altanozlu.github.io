+++
title = "Using RTC to run services with power savings!"
date = 2025-02-26
+++

For some time, I've been using my desktop to run some applications, some of which are written by me and it's ideal to run it every 30 minutes or so.
Since it's an AMD machine its idle power usage is not ideal.
It's around `90W` without monitor, I couldn't get it lower with power saving settings in BIOS so I just gave up.
A few days ago I wanted to know if it's possible to wake the machine programmatically and came across rtcwake.

So I just wrote a small Rust script around it,

When you call `suspend_until` it will just sleep, set RTC and continue until machine is suspended by external factors (like me!)
if machine wakes up from suspend because of RTC, next time `suspend_until` called it will also suspend the machine after setting RTC.
This cycle can be broken by external factor wake the machine before RTC.

This way machine will only spend `1W` when it's suspended. It is also possible to set other modes for RTC, check [man rtcwake](https://www.man7.org/linux/man-pages/man8/rtcwake.8.html). 





How to use rtcwake (run this as root):

```rust
let mut rtc = rtcwake::RTCWake::new(Duration::from_minutes(30));
loop {
  println!("hey!");
  rtc.suspend_until()?;
}
````

rtcwake module:

```rust
use std::{
    error::Error,
    process::Command,
    time::{Duration, SystemTime},
};

const TICK_SECONDS: Duration = Duration::from_secs(5);

enum Status {
    Awake,
    Suspend,
}

pub struct RTCWake {
    status: Status,
    duration: Duration,
}

impl RTCWake {
    pub fn new(duration: Duration) -> Self {
        RTCWake {
            status: Status::Awake,
            duration,
        }
    }
    pub fn suspend_until(&mut self) -> Result<(), Box<dyn Error>> {
        let desired_wake_time = SystemTime::now() + self.duration;
        match self.status {
            Status::Awake => self.sleep(self.duration)?,
            Status::Suspend => {
                call_rtcwake("mem", self.duration)?;
                let now = SystemTime::now();
                if now < desired_wake_time {
                    // System woke up before RTC
                    println!("{now:?} < {desired_wake_time:?}");
                    let sleep_duration = desired_wake_time.duration_since(now)?;
                    self.status = Status::Awake;
                    self.sleep(sleep_duration)?;
                }
            }
        }
        Ok(())
    }

    fn sleep(&mut self, duration: Duration) -> Result<(), Box<dyn Error>> {
        let desired_wake_time = SystemTime::now() + duration;
        call_rtcwake("no", duration)?; // Now it won't suspend but just in case it will set up RTC.
        loop {
            let time_before_sleep = SystemTime::now();
            std::thread::sleep(TICK_SECONDS);
            let passed_time = SystemTime::now().duration_since(time_before_sleep)?;
            if passed_time > TICK_SECONDS * 5 {
                // Thread slept more than it should, assuming RTC wake
                self.status = Status::Suspend;
                println!("Woke with RTC");
            }
            if SystemTime::now() >= desired_wake_time {
                break;
            }
        }
        Ok(())
    }
}

fn call_rtcwake(mode: &str, duration: Duration) -> Result<(), Box<dyn Error>> {
    Command::new("/usr/bin/rtcwake")
        .arg("-m")
        .arg(mode)
        .arg("-s")
        .arg(duration.as_secs().to_string())
        .output()?;
    Ok(())
}
```
