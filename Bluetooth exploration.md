Seems like the crunchiness is because it takes time to grant priority. Initially pipewire runs at normal priority, causing crackling then RTKit elevates it to -11 priority. Claude is helping me test this assumption based on my reading.

```sh
~ ❯ ps -elf | grep pipewire
0 S kadan       1368     747 12  69 -11 - 27180 do_epo 17:33 ?        00:06:17 /usr/bin/pipewire
0 S kadan       1413     747  0  80   0 - 27856 do_epo 17:33 ?        00:00:25 /usr/bin/pipewire-pulse
0 S kadan      10504    3413  0  80   0 -  1716 anon_p 18:23 pts/0    00:00:00 grep pipewire

~ ❯ systemctl --user status pipewire pipewire-pulse
● pipewire.service - PipeWire Multimedia Service
     Loaded: loaded (/usr/lib/systemd/user/pipewire.service; disabled; preset: enabled)
     Active: active (running) since Thu 2025-08-28 17:33:04 EDT; 50min ago
 Invocation: 51a763a82edf437e9d6bfbbb9e4f8102
TriggeredBy: ● pipewire.socket
   Main PID: 1368 (pipewire)
      Tasks: 3 (limit: 9019)
     Memory: 11.2M (peak: 12.1M)
        CPU: 6min 17.157s
     CGroup: /user.slice/user-1000.slice/user@1000.service/session.slice/pipewire.service
             └─1368 /usr/bin/pipewire

Aug 28 17:33:04 omarchy-intel-macbook systemd[747]: Started PipeWire Multimedia Service.

● pipewire-pulse.service - PipeWire PulseAudio
     Loaded: loaded (/usr/lib/systemd/user/pipewire-pulse.service; disabled; preset: enabled)
     Active: active (running) since Thu 2025-08-28 17:33:05 EDT; 50min ago
 Invocation: 9dda3babe5284bcd992ea22209150ece
TriggeredBy: ● pipewire-pulse.socket
   Main PID: 1413 (pipewire-pulse)
      Tasks: 3 (limit: 9019)
     Memory: 14.8M (peak: 16M)
        CPU: 25.795s
     CGroup: /user.slice/user-1000.slice/user@1000.service/session.slice/pipewire-pulse.service
             └─1413 /usr/bin/pipewire-pulse


~ ✗ journalctl | grep -E "(rtkit|nice-level)" | tail -10
Aug 28 17:33:04 omarchy-intel-macbook rtkit-daemon[1364]: Supervising 2 threads of 1 processes of 1 users.
Aug 28 17:33:04 omarchy-intel-macbook rtkit-daemon[1364]: Supervising 2 threads of 1 processes of 1 users.
Aug 28 17:33:04 omarchy-intel-macbook rtkit-daemon[1364]: Supervising 2 threads of 1 processes of 1 users.
Aug 28 17:33:04 omarchy-intel-macbook rtkit-daemon[1364]: Successfully made thread 1369 of process 1369 owned by '1000' high priority at nice level -11.
Aug 28 17:33:04 omarchy-intel-macbook rtkit-daemon[1364]: Supervising 3 threads of 2 processes of 1 users.
Aug 28 17:33:04 omarchy-intel-macbook rtkit-daemon[1364]: Successfully made thread 1381 of process 1369 owned by '1000' RT at priority 20.
Aug 28 17:33:04 omarchy-intel-macbook rtkit-daemon[1364]: Supervising 4 threads of 2 processes of 1 users.
Aug 28 17:33:04 omarchy-intel-macbook rtkit-daemon[1364]: Supervising 4 threads of 2 processes of 1 users.
Aug 28 17:33:04 omarchy-intel-macbook rtkit-daemon[1364]: Supervising 4 threads of 2 processes of 1 users.
Aug 28 17:33:04 omarchy-intel-macbook rtkit-daemon[1364]: Supervising 4 threads of 2 processes of 1 users.

~ ❯ journalctl --user -u pipewire-pulse | grep -E "(rtkit|nice|priority)" | tail -5

~ ❯ pactl list sinks | grep -A 20 bluez
	Name: bluez_output.88_C9_E8_6B_DE_DF.1
	Description: WH-1000XM5
	Driver: PipeWire
	Sample Specification: s16le 2ch 48000Hz
	Channel Map: front-left,front-right
	Owner Module: 4294967295
	Mute: no
	Volume: front-left: 39218 /  60% / -13.38 dB,   front-right: 39218 /  60% / -13.38 dB
	        balance 0.00
	Base Volume: 65536 / 100% / 0.00 dB
	Monitor Source: bluez_output.88_C9_E8_6B_DE_DF.1.monitor
	Latency: 0 usec, configured 0 usec
	Flags: HARDWARE HW_VOLUME_CTRL DECIBEL_VOLUME LATENCY
	Properties:
		api.bluez5.address = "88:C9:E8:6B:DE:DF"
		api.bluez5.codec = "aac"
		api.bluez5.profile = "a2dp-sink"
		api.bluez5.transport = ""
		bluez5.loopback = "false"
		card.profile.device = "1"
		device.id = "93"
		device.routes = "1"
		factory.name = "api.bluez5.a2dp.sink"
		device.description = "WH-1000XM5"
		node.name = "bluez_output.88_C9_E8_6B_DE_DF.1"
		node.pause-on-idle = "false"
		priority.driver = "1010"
		priority.session = "1010"
		spa.object.id = "1"
		factory.id = "12"
		clock.quantum-limit = "8192"
		device.api = "bluez5"
		media.class = "Audio/Sink"
		media.name = "WH-1000XM5"
		node.driver = "true"
		port.group = "stream.0"
		node.loop.name = "data-loop.0"
		library.name = "audioconvert/libspa-audioconvert"
		object.id = "101"
		object.serial = "177"
		client.id = "42"
		api.bluez5.class = "0x240404"
		api.bluez5.connection = "connected"
		api.bluez5.device = ""
		api.bluez5.icon = "audio-headset"
		api.bluez5.path = "/org/bluez/hci0/dev_88_C9_E8_6B_DE_DF"
		bluez5.profile = "off"
		device.alias = "WH-1000XM5"
		device.bus = "bluetooth"
		device.form_factor = "headset"
		device.icon_name = "audio-headset-bluetooth"
		device.name = "bluez_card.88_C9_E8_6B_DE_DF"
		device.product.id = "0x0df0"
		device.string = "88:C9:E8:6B:DE:DF"
		device.vendor.id = "usb:054c"
	Ports:
		headset-output: Headset (type: Headset, priority: 0, available)
	Active Port: headset-output
	Formats:
		pcm

~ ❯ journalctl --user -u pipewire | grep "client too slow" | tail -3
```

Testing with: Sony WH-1000XM5 using **AAC codec** via A2DP profile. No client too slow errors.

The **pipewire-pulse** process (PID 1413) is running at normal priority (nice 0) instead of elevated priority (nice -11). This process specifically handles PulseAudio compatibility and Bluetooth audio routing. When it starts, it can't keep up with the audio stream initially (causing crunchiness), but eventually the system buffers compensate and it smooths out.

Going to enable RTKit on boot so the stuttering does not happen again.