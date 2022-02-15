# rusty-daw-io Design Document

*(note "rusty-daw-io" may not be the final name of this crate)*

# Objective

The goal of this crate is to provide a powerful, cross-platform, highly configurable, low-latency, and robust solution for connecting audio software to audio and MIDI devices.

## Why not contribute to an already existing project like `RTAudio` or `CPAL`?

### RTAudio
- This API is written in a complicated C++ codebase, making it very tricky to bind to other languages such as Rust.
- This project has a poor track record in its stability and ability to gracefully handle errors (not ideal for live audio software).

### CPAL
In short, CPAL is very opinionated, and we have a few deal-breaking issues with its core design.

- CPAL's design does not handle duplex audio devices well. It spawns each input and output stream into separate threads, requiring the developer to sync them together with ring buffers. This is inneficient for most consumer and professional duplex audio devices which already have their inputs and outputs tied into the same stream to reduce latency.
- The API for searching for and configuring audio devices is cumbersome. It returns a list of every possible combination of configurations available with the system's devices. This is not how a user configuring audio settings through a GUI expects this to work.
- CPAL does not have any support for MIDI devices, so we would need to write our own support for it anyway.

Why not just fork `CPAL`?
- To fix these design issues we would pretty much need to rewrite the whole API anyway. Of course we don't have to work completely from scratch. We can still borrow some of the low-level platform specific code in CPAL.

# Goals
- Support for Linux, Mac, and Windows using the following backends: (and maybe Android and iOS in the future, but that is not a gaurantee)
    - Linux
        - [ ] Jack
        - [ ] Pipewire
        - [ ] Alsa (Maybe, depending on how difficult this is. This could be unecessary if Pipewire turns out to be good enough.)
        - [ ] Portaudio (Maybe, depending on how difficult this is. This could be unecessary if Pipewire turns out to be good enough.)
    - Mac
        - [ ] CoreAudio
        - [ ] Jack (Maybe, if it is stable enough on Mac.)
        - [ ] Portaudio (Maybe, if it is stable enough on Mac.)
    - Window
        - [ ] WASAPI
        - [ ] ASIO (reluctantly) (If there is a better solution, let me know because I would rather avoid this proprietary mess.)
        - [ ] Jack (Maybe, if it is stable enough on Windows.)
        - [ ] Portaudio (Maybe, if it is stable enough on Windows.)
- Scan the available devices on the system, and present configuration options in a format that is intuitive to an end-user configuring devices inside a settings GUI.
- Send all audio and midi streams into a single high-priority thread, taking advantage of native duplex devices when available. (Audio buffers will be presented as de-interlaced `f32` buffers).
- Robust and graceful error handling, especially while the stream is running.
- Save and load configurations to/from a config file.
- Let the user create and name "busses" which are presented in the application's GUI and config files (I.e. showing the ports "focusrite-scarlett:input_1" and "focusrite-scarlett:input_2" as busses named "host mic" and "co-host mic", and showing the ports "focusrite-scarlett:output_1" and "focusrite-scarlett:output_2" as a single stereo bus named "speakers out"). This could arguably be handled by the application instead, but this could allow config files to be shared more easily, even across different applications that use `rusty-daw-io`.
- A system that will try to automatically create a good initial default configuration.

# Later/Maybe Goals
- Support MIDI 2.0 devices
- Support for OSC devices
- C API bindings

# Non-Goals
- No Android and iOS support (for now atleast)
- No support for using multiple backends at the same time (i.e trying to use WASAPI device as an input and an ASIO device as an output). This will just add a whole slew of complexity and stuff that can go wrong.
- No support for tying multiple separate (non-duplexed) audio devices together. We will only support either connecting to a single duplex audio device *or* connecting to a single non-duplex output device.
    - This one is probably controversal, so let me explain the reasoning:
        - Pretty much all modern external audio devices (a setup used by most professionals and pro-sumers) are already duplex.
        - MacOS (and in Linux using JACK or Pipewire) already packages all audio device streams into a single "system-wide duplex device". So this is really only a Windows-specific problem.
        - Tying together multiple non-duplex audio streams requires an intermediate buffer that adds a sometimes unkowable amount of latency.
        - Allowing for multiple separate audio devices adds a lot of complexity to both the settings GUI and the config file, and a lot more that can go wrong.
        - Some modern DAWs like Bitwig already use this "single audio device only" system, so it's not like it's a new concept.
- No support for non-f32 audio streams.
    - There is just no point in my opinion in presenting any other sample format other than `f32` in such an API. These `f32` buffers will just be converted to/from the native sample format that the device wants behind the scenes.

# API Design

### ***Please note that the current code in this repo is outdated.***

The API is divided into three stages: Enumerating the available devices, creating a config, and running the stream.

## Device Enumeration API:

```rust
// The user calls these to retrieve the list of available audio backends
// and MIDI backends on the system. (i.e. Jack, Pipewire, WASAPI, CoreAudio, etc.)
//
// Calling these a second time will essentially "refresh" the list of available
// devices.
pub fn audio_backends() -> &[AudioBackendInfo] {
    // Wrapping the underlying platform-specific code will work like this:
    #[cfg(target_os = "linux")]
    linux::audio_backends()

    #[cfg(target_os = "macos")]
    macos::audio_backends()

    #[cfg(target_os = "windows")]
    windows::audio_backends()
}
pub fn midi_backends() -> &[MidiBackendInfo] {
    #[cfg(target_os = "linux")]
    linux::midi_backends()

    #[cfg(target_os = "macos")]
    macos::midi_backends()

    #[cfg(target_os = "windows")]
    windows::midi_backends()
}

pub struct AudioBackendInfo {
    /// The name of this backend (i.e. Jack, Pipewire, WASAPI, CoreAudio, etc.)
    pub name: String,

    /// If true, then it means this backend is the default/preferred backend for
    /// the given system. Only one item in the `audio_backends()` list will have
    /// this set to true.
    pub is_default: bool,

    /// The version of this backend (if there is one)
    pub version: Option<String>,

    /// The devices that are available in this backend.
    /// 
    /// Please note that these are not necessarily each physical device in the
    /// system. For example, in backends like Jack and CoreAudio, the whole system
    /// acts like a single "duplex device" which is the audio server.
    pub devices: Vec<AudioDeviceInfo>,

    /// If this is true, then it means it is relevant to actually show the available
    /// devices as a list to select from in a settings GUI.
    /// 
    /// In backends like Jack and CoreAudio which set this to false, there is only
    /// ever one "systemwide duplex device" which is the audio server, and showing
    /// this information in a settings GUI is irrelevant.
    pub devices_are_relevant: bool,

    /// If this is true, then it means this backend is available and running on
    /// this system. (For example, if this backend is Jack and the Jack server is
    /// not currently running on the system, then this will be false.)
    pub available: bool,
}

/// The info about a particular audio device.
pub struct AudioDeviceInfo {
    /// The name of this device.
    /// 
    /// Note if there are multiple devices with the same name then a number should
    /// be appended to it. (i.e. "Interface, "Interface #2")
    pub name: String,

    /// If true, then it means this device is the default/preferred device for
    /// the given backend. Only one device in the backend's list will have this set
    /// to true.
    pub is_default: bool,

    /// The names of the available input ports (one port per channel) on this device
    /// (i.e. "mic_1", "mic_2", "system_input", etc.)
    pub in_ports: Vec<String>,

    /// The names of the available output ports (one port per channel) on this device
    /// (i.e. "out_1", "speakers_out_left", "speakers_out_right", etc.)
    pub out_ports: Vec<String>,

    /// The available sample rates for this device.
    pub sample_rates: Vec<u32>,

    /// The default/preferred sample rate for this audio device.
    pub default_sample_rate: u32,

    /// The supported range of fixed buffer/block sizes for this device. If the device
    /// doesn't support fixed-size buffers then this will be `None`.
    pub buffer_size_range: Option<FixedBufferSizeRange>,

    /// The indexes of the default/preferred input ports for this audio device.
    pub default_in_ports: DefaultChannelLayout,

    /// The indexes of the default/preferred output ports for this audio device.
    pub default_out_ports: DefaultChannelLayout,
}

pub struct FixedBufferSizeRange {
    // The minimum block size (inclusive)
    pub min: u32,
    // The maximum block size (inclusive)
    pub max: u32,

    // If this is `true` then that means the device only supports fixed block sizes
    // between `min` and `max` that are a power of 2.
    pub must_be_power_of_2: bool,

    // The default/preferred fixed buffer size for this device.
    pub default: u32,
}

// These contain the "indexes" of the ports assigned to each channel.
pub enum DefaultChannelLayout {
    Unspecified,
    Mono(usize),
    Stereo { left: usize, right: usize },
    Surround51 { center: usize, left: usize, right: usize, y1: usize, y2: usize },
    ...
}

pub struct MidiBackendInfo {
    // The name of this backend (i.e. Jack, Pipewire, WASAPI, CoreAudio, etc.)
    pub name: String,

    // If true, then it means this backend is the default/preferred backend for
    // the given system. Only one item in the `audio_backends()` list will have
    // set this to true.
    pub is_default: bool,

    // The version of this backend (if there is one)
    pub version: Option<String>,

    // The list of available input MIDI devices
    pub in_devices: Vec<MidiDeviceInfo>,

    // The list of available output MIDI devices
    pub out_devices: Vec<MidiDeviceInfo>,

    // If this is true, then it means this backend is available and running on
    // this system. (For example, if this backend is Jack and the Jack server is
    // not currently running on the system, then this will be false.)
    pub available: bool,
}

pub struct MidiDeviceInfo {
    // The name of this device
    pub name: String,

    // If true, then it means this device is the default/preferred device for
    // the given backend. Only one input and one output device in the backend's
    // list will have this set to true.
    pub is_default: bool,
}
```

## Configuration API:

This is the API for the "configuration". The user constructs this configuration in whatever method they choose (from a settings GUI or a config file) and sends it to this crate to be ran.

```rust
pub struct Config {
    /// The name of the audio backend to use.
    /// 
    /// Set this to `None` to automatically select the default backend for the system.
    pub audio_backend: Option<String>,

    /// The name of the audio device to use.
    /// 
    /// Set this to `None` to automatically select the default device for the backend.
    pub audio_device: Option<String>,

    /// The audio input busses to create/use from the device's ports. These are the "internal" busses
    /// that appear to the user as list of available sources/sends in the application. This is not
    /// necessarily the same as the actual names of the ports on the device.
    /// 
    /// Set this to `None` to automatically select the default port layout for the device.
    pub audio_in_busses: Option<Vec<AudioBusConfig>>,

    /// The audio output busses to create/use from the device's ports. These are the "internal" busses
    /// that appear to the user as list of available sources/sends in the application. This is not
    /// necessarily the same as the actual names of the ports on the device.
    /// 
    /// Set this to `None` to automatically select the default port layout for the device.
    pub audio_out_busses: Option<Vec<AudioBusConfig>>,

    /// The sample rate to use.
    ///
    /// Set this to `None` to use the default sample rate of the system device.
    pub sample_rate: Option<u32>,
    
    /// The fixed buffer size to user for this audio device.
    ///
    /// Set this to `None` to use the default settings of the system device.
    pub fixed_buffer_size: Option<u32>,

    /// The configuration for MIDI devices.
    /// 
    /// Set this to `None` to use no MIDI devices in the stream.
    pub midi_config: Option<MidiConfig>,
}

pub struct MidiConfig {
    /// The name of the MIDI backend to use.
    /// 
    /// Set this to `None` to automatically select the default backend for the system.
    pub backend: Option<String>,

    /// The midi input controllers to use.
    /// 
    /// Set this to `None` to use the default inputs for the backend.
    pub in_controllers: Option<Vec<MidiControllerConfig>>,

    /// The midi output controllers to use.
    /// 
    /// Set this to `None` to use the default outputs for the backend.
    pub out_controllers: Option<Vec<MidiControllerConfig>>,
}

pub struct AudioBusConfig {
    /// The ID to use for this bus. This ID is for the "internal" bus that appears to the user
    /// as list of available sources/sends in the application. This is not necessarily the same
    /// as the name of the actual ports of the device.
    ///
    /// Examples of IDs can include:
    ///
    /// * Realtek Device In
    /// * Drums Mic
    /// * Headphones Out
    /// * Speakers Out
    pub id: String,

    /// The ports (of the system device) that this bus will be connected to. The buffers presented
    /// in the process() thread will appear in this same order.
    pub system_ports: Vec<String>,
}

pub struct MidiControllerConfig {
    /// The ID to use for this controller. This ID is for the "internal" controller that appears to
    /// the user as list of available sources/sends in the application. This is not necessarily the
    /// same as the name of the actual system hardware device that this "internal" controller is
    /// connected to.
    /// 
    /// Set this to `None` to use the system device name for this controller.
    pub id: String,

    /// The name of the MIDI device this controller is connected to.
    pub system_device: String,
}
```

## Running API:

The user sends a config to this API to run it.

```rust
// The user can call this to get the estimated total latency of a particular
// configuration before running it.
//
// `None` will be returned if the latency is not known at this time.
pub fn estimated_latency(config: &Config) -> Option<u32> {
    ...
}

// The user calls this to get the sample rate of a particular configuration
// before running it.
//
// `None` will be returned if the sample rate is not known at this time.
pub fn sample_rate(config: &Config) -> Option<u32> {
    ...
}

// The user derives this trait for their own custom struct. These methods get called in
// the `run()` method.
pub trait ProcessHandler: 'static + Send {
    /// Initialize/allocate any buffers here. This will only be called once
    /// on creation.
    fn init(&mut self, stream_info: &StreamInfo);

    /// This gets called if the user made a change to the configuration that does not
    /// require restarting the audio thread.
    fn stream_changed(&mut self, stream_info: &StreamInfo);

    /// Process the current buffers. This will always be called on a realtime thread.
    fn process(&mut self, proc_info: ProcessInfo);
}

// TODO: API of `StreamInfo` and `ProcessInfo`.

pub trait ErrorHandler: 'static + Send + Sync {
    /// Called when a non-fatal error occurs (any error that does not require the audio
    /// thread to restart).
    fn nonfatal_error(&mut self, error: StreamError);

    /// Called when a fatal error occurs (any error that requires the audio thread to
    /// restart).
    fn fatal_error(self, error: FatalStreamError);
}

// ---- Run Options -------------------------------------------------------------------

// These are flags passed into the `run()` method that describe how the user wants the
// stream to behave to certain errors.

// Note the API of this section is still a work in progress. We will add/remove items
// as we deem necessary.

pub struct RunOptions {
    // If Some, then the backend will use this name as the client name that appears
    // in the audio server. This is only relevent for some backends like Jack.
    use_application_name: Option<String>,

    audio_backend_not_found: NotFoundBehavior,
    audio_device_not_found: NotFoundBehavior,
    audio_bus_config_error: AudioBusConfigErrorBehavior,
    buffer_size_config_error: BufferSizeConfigErrorBehavior,

    midi_backend_not_found: NotFoundBehavior,
    midi_device_not_found: MidiDeviceNotFoundError,
}

pub enum NotFoundBehavior {
    ReturnWithError,
    TryNextBest,
}

pub enum AudioBusConfigErrorBehavior {
    ReturnWithError,
    DiscardInvalidBusses,
    DiscardAllBussesAndTryDefaults,
}

pub enum SampleRateConfigErrorBehavior {
    ReturnWithError,
    TryNextBest,
    TryNextBestWithMinMaxSR((u32, u32)),
}

pub enum BufferSizeConfigErrorBehavior {
    ReturnWithError,
    TryNextBest,
    TryNextBestWithMinMaxSize((u32, u32)),
}

pub enum MidiDeviceNotFoundError {
    ReturnWithError,
    DiscardInvalidControllers,
    DiscardAllControllersAndTryDefault,
}

// ------------------------------------------------------------------------------------

// Run the given config in an audio thread.
pub fn run<P: ProcessHandler, E: ErrorHandler>((
    config: &Config,
    options: &RunOptions,
    process_handler: P,
    error_handler: E,
) -> Result<StreamHandle<P, E>, RunConfigError> {
    ...
}

// This struct contains a handle to the actual stream.
//
// When this gets dropped, the stream should also automatically stop. This is the
// intended way for the user to stop a stream.
pub struct StreamHandle<P: ProcessHandler, E: ErrorHandler> {
    #[cfg(target_os = "linux")]
    os_handle: LinuxStreamHandle<P, E>,

    #[cfg(target_os = "macos")]
    os_handle: MacOSStreamHandle<P, E>,

    #[cfg(target_os = "windows")]
    os_handle: WindowsStreamHandle<P, E>,
}

impl<P: ProcessHandler, E: ErrorHandler> StreamHandle<P, E> {
    // Returns the actual configuration of the running stream.
    pub fn stream_info(&self) -> &StreamInfo {
        ...
    }

    // The user can call this to change the audio bus configuration while the
    // audio thread is still running. Support for this will depend on the
    // backend.
    //
    // If the given config is invalid, an error will be returned with no
    // effect on the running audio thread.
    pub fn change_audio_bus_config(
        &mut self,
        audio_in_busses: Option<Vec<AudioBusConfig>>,
        audio_out_busses: Option<Vec<AudioBusConfig>>
    ) -> Result<(), ChangeAudioBusConfigError> {
        ...
    }

    // The user can call this to change the buffer size configuration while the
    // audio thread is still running. Support for this will depend on the
    // backend.
    //
    // If the given config is invalid, an error will be returned with no
    // effect on the running audio thread.
    pub fn change_buffer_size_config(
        &mut self,
        buffer_size: Option<u32>,
    ) -> Result<(), ChangeBufferSizeConfigError> {
        ...
    }

    // Returns whether or not this backend supports changing the audio bus
    // configuration while the audio thread is running.
    pub fn can_change_audio_bus_config(&self) -> bool {
        ...
    }

    // Returns whether or not this backend supports changing the buffer size
    // configuration while the audio thread is running.
    pub fn can_change_buffer_size_config(&self) -> bool {
        ...
    }
}
```

# More Stuff

In addition to the main API, we will also have a full-working demo application with a working settings GUI. This will probably be written in `egui`, but another UI toolkit could be used.

We may also consider creating a standard for a config file that can be shared across different applications which use `rusty-daw-io`. This will likely use the TOML file format.
