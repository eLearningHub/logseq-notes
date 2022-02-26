- > Zig is a general-purpose programming language and toolchain for maintaining robust, optimal, and reusable software.
-
- Zig 是一门刚诞生的语言，核心特性包括
	- 简单：没有隐式逻辑，隐式分配，没有预处理，没有宏
	- 基于编译时代码执行和惰性分析做元编程 (不太懂)
	- 完整的工具链
		- 能够直接 replace C/C++，原生支持交叉编译
		- 提供 `zig build` 来构建跨平台的二进制
		- 能够跟 C/C++ 无缝集成
-
- 比较核心的优势
	- 跟 C 的无缝集成
		- Integration with C libraries without FFI/bindings
			- `zig` 本身就是一个 C 编译器(via [[LLVM]])
		- 可以直接引入一个 C `.h`
collapsed:: true
			- ```zig
			  const c = @cImport(@cInclude("soundio/soundio.h"));
			  const std = @import("std");
			  
			  fn sio_err(err: c_int) !void {
			      switch (@intToEnum(c.SoundIoError, err)) {
			          .None => {},
			          .NoMem => return error.NoMem,
			          .InitAudioBackend => return error.InitAudioBackend,
			          .SystemResources => return error.SystemResources,
			          .OpeningDevice => return error.OpeningDevice,
			          .NoSuchDevice => return error.NoSuchDevice,
			          .Invalid => return error.Invalid,
			          .BackendUnavailable => return error.BackendUnavailable,
			          .Streaming => return error.Streaming,
			          .IncompatibleDevice => return error.IncompatibleDevice,
			          .NoSuchClient => return error.NoSuchClient,
			          .IncompatibleBackend => return error.IncompatibleBackend,
			          .BackendDisconnected => return error.BackendDisconnected,
			          .Interrupted => return error.Interrupted,
			          .Underflow => return error.Underflow,
			          .EncodingString => return error.EncodingString,
			          else => return error.Unknown,
			      }
			  }
			  
			  var seconds_offset: f32 = 0;
			  
			  fn write_callback(
			      maybe_outstream: ?[*]c.SoundIoOutStream,
			      frame_count_min: c_int,
			      frame_count_max: c_int,
			  ) callconv(.C) void {
			      const outstream = @ptrCast(*c.SoundIoOutStream, maybe_outstream);
			      const layout = &outstream.layout;
			      const float_sample_rate = outstream.sample_rate;
			      const seconds_per_frame = 1.0 / @intToFloat(f32, float_sample_rate);
			      var frames_left = frame_count_max;
			  
			      while (frames_left > 0) {
			          var frame_count = frames_left;
			  
			          var areas: [*]c.SoundIoChannelArea = undefined;
			          sio_err(c.soundio_outstream_begin_write(
			              maybe_outstream,
			              @ptrCast([*]?[*]c.SoundIoChannelArea, &areas),
			              &frame_count,
			          )) catch |err| std.debug.panic("write failed: {s}", .{@errorName(err)});
			  
			          if (frame_count == 0) break;
			  
			          const pitch = 440.0;
			          const radians_per_second = pitch * 2.0 * std.math.pi;
			          var frame: c_int = 0;
			          while (frame < frame_count) : (frame += 1) {
			              const sample = std.math.sin((seconds_offset + @intToFloat(f32, frame) *
			                  seconds_per_frame) * radians_per_second);
			              {
			                  var channel: usize = 0;
			                  while (channel < @intCast(usize, layout.channel_count)) : (channel += 1) {
			                      const channel_ptr = areas[channel].ptr;
			                      const sample_ptr = &channel_ptr[@intCast(usize, areas[channel].step * frame)];
			                      @ptrCast(*f32, @alignCast(@alignOf(f32), sample_ptr)).* = sample;
			                  }
			              }
			          }
			          seconds_offset += seconds_per_frame * @intToFloat(f32, frame_count);
			  
			          sio_err(c.soundio_outstream_end_write(maybe_outstream)) catch |err| std.debug.panic("end write failed: {s}", .{@errorName(err)});
			  
			          frames_left -= frame_count;
			      }
			  }
			  
			  pub fn main() !void {
			      const soundio = c.soundio_create();
			      defer c.soundio_destroy(soundio);
			  
			      try sio_err(c.soundio_connect(soundio));
			  
			      c.soundio_flush_events(soundio);
			  
			      const default_output_index = c.soundio_default_output_device_index(soundio);
			      if (default_output_index < 0) return error.NoOutputDeviceFound;
			  
			      const device = c.soundio_get_output_device(soundio, default_output_index) orelse return error.OutOfMemory;
			      defer c.soundio_device_unref(device);
			  
			      std.debug.print("Output device: {s}\n", .{device.*.name});
			  
			      const outstream = c.soundio_outstream_create(device) orelse return error.OutOfMemory;
			      defer c.soundio_outstream_destroy(outstream);
			  
			      outstream.*.format = @intToEnum(c.SoundIoFormat, c.SoundIoFormatFloat32NE);
			      outstream.*.write_callback = write_callback;
			  
			      try sio_err(c.soundio_outstream_open(outstream));
			  
			      try sio_err(c.soundio_outstream_start(outstream));
			  
			      while (true) c.soundio_wait_events(soundio);
			  }
			  ```
-
- 样例
	- ```zig
	  const std = @import("std");
	  const json = std.json;
	  const payload =
	      \\{
	      \\    "vals": {
	      \\        "testing": 1,
	      \\        "production": 42
	      \\    },
	      \\    "uptime": 9999
	      \\}
	  ;
	  const Config = struct {
	      vals: struct { testing: u8, production: u8 },
	      uptime: u64,
	  };
	  const config = x: {
	      var stream = json.TokenStream.init(payload);
	      const res = json.parse(Config, &stream, .{});
	      // Assert no error can occur since we are
	      // parsing this JSON at comptime!
	      break :x res catch unreachable;
	  };
	  pub fn main() !void {
	      if (config.vals.production > 50) {
	          @compileError("only up to 50 supported");
	      }
	      std.log.info("up={d}", .{config.uptime});
	  }
	  ```