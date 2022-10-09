# ESP-MAX7219-NOSTD :crab:

This crate is created on basis of [Philipp Schuster's](https://github.com/phip1611/max-7219-led-matrix-util) crate.

## Description
This is `no-std` crate to make `max7219 LED Matrix Display` work with multiple Espressif Chips and do some basic operations (new features will be added over time). 
#### Cargo.toml
```toml
[dependencies]
esp-max7219-nostd = { version = "0.1.0", git = "https://github.com/playfulFence/esp-max7219-nostd" }
```

## Usage example (ESP32-C3)
```rust
#![no_std]
#![no_main]

use esp32c3_hal::{
    adc::{AdcConfig, Attenuation, ADC, ADC1},
    clock::ClockControl,
    pac::Peripherals,
    gpio_types::*,
    gpio::*,
    prelude::*,
    spi,
    timer::TimerGroup,
    Rtc,
    IO,
    Delay,
};

use max7219::connectors::PinConnector;
use max7219::MAX7219;
use max7219::DecodeMode;

use esp_max7219_nostd::{prepare_display, show_moving_text_in_loop};

use riscv_rt::entry; // for C3 chip
use esp_backtrace as _;

extern crate alloc;
#[global_allocator]  // necessary for correct work of alloc on ESP chips
static ALLOCATOR: esp_alloc::EspHeap = esp_alloc::EspHeap::empty();

fn init_heap() {
    const HEAP_SIZE: usize = 32 * 1024;

    extern "C" {
        static mut _heap_start: u32;
    }

    unsafe {
        let heap_start = &_heap_start as *const _ as usize;
        ALLOCATOR.init(heap_start as *mut u8, HEAP_SIZE);
    }
}

#[entry]
fn main() -> ! {
    init_heap(); // for correct allocation
    let peripherals = Peripherals::take().unwrap();

    let mut system = peripherals.SYSTEM.split();

    let mut clocks = ClockControl::boot_defaults(system.clock_control).freeze();

    // Disable the RTC and TIMG watchdog timers
    let mut rtc = Rtc::new(peripherals.RTC_CNTL);
    let timer_group0 = TimerGroup::new(peripherals.TIMG0, &clocks);
    let mut wdt0 = timer_group0.wdt;
    let timer_group1 = TimerGroup::new(peripherals.TIMG1, &clocks);
    let mut wdt1 = timer_group1.wdt;
    
    rtc.rwdt.disable();
    wdt0.disable();
    wdt1.disable();


    // println!("About to initialize the SPI LED driver ILI9341");
    let io = IO::new(peripherals.GPIO, peripherals.IO_MUX);
    
    let mut delay = Delay::new(&clocks);

    let din = io.pins.gpio7.into_push_pull_output();
    let cs = io.pins.gpio3.into_push_pull_output();
    let clk = io.pins.gpio6.into_push_pull_output();

    let mut display = MAX7219::from_pins(1, din, cs, clk).unwrap(); // replace "1" with number of displays in chain, if you have more
    prepare_display(&mut display, 1, 0x5);
    show_moving_text_in_loop(
        &mut display, 
        "Hello, Espressif",
        1, // replace "1" with number of displays in chain, if you have more
        30, 
        2, 
        &mut delay,
    );
    loop {

    }
}
```
