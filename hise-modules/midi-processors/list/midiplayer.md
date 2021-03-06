---
keywords: Midi Player
summary:  A MIDI processor that plays MIDI files.
author:   Christoph Hart
modified: 18.03.2019
---

The **MIDI Player** is a generic MIDI file player that allows you to play MIDI loops in your project, record incoming MIDI notes and provides the tools for processing MIDI files on the fly. 

## List of supported MIDI features

The MIDI player in its current state is not intended to be a 100% standard compliant MIDI player - it's purpose is to be a player for MIDI loops that are synced to an external clock. That implies that on every NoteOff the MIDI time information (song tempo, tempo changes, time signatures and other meta events) get removed. There's no support for CC controllers or pitch wheel information, either, but these are just not implemented yet because of laziness so if you need it, drop a post in the HISE forum and I'll put it on my queue..

## Using the MIDI player

If you add a **MIDI Player** to the MIDI chain of a Sound Generator you will notice that it'll jump to the top of the chain. I will spare you the technical explanation of why this is necessary, but be aware of this fact. If you need to "preprocess" events that should go into the MIDI player, you need to put the preprocessor in its parent's MIDI chain.

Drop some MIDI files into the box that says **Drop MIDI files** and press the play button to listen to the MIDI file in a loop.

After this "Hello World experience", you might want to dive into the MIDI Players more complex capabilities, which are covered in this chapter.

## MidiFile Pool

The MIDI files that you load into the MIDI Player module will be [pooled](/working-with-hise/project-management#file-pools) like AudioFiles, Images and SampleMaps so that they can be embedded into the compiled binary. All files in the [MidiFiles](/working-with-hise/project-management/projects-folders/midi-files) project folder will be cached on project load. The MIDI files size is basically neglible, unlike samples or images, so you won't need to call something like `Engine.loadAudioFilesIntoPool()` explicitely, to pool them.

You can hotswap different MIDI files during playback and the player will try to keep the same playback position as good as possible (by wrapping the playback position around the new loop length if it's shorter than the previous file).

### MIDI tracks

A MIDI file can contain up to 16 MIDI tracks, however the MIDI Player can just play one track at a time. If you want to play multiple tracks, you'll have to create another MIDI Player instance in another Sound Generator (because chances are great you will want to play it with another sound) and change the MIDI files track index). Be aware that the track index is supposed to be a static property so hotswapping between different tracks is not supported (unlike hotswapping files).

## MIDI Player UI

If you load up the MIDI player, it will look pretty unimpressive. The reason is that the core player only handles the basic playback functionality and all other UI features are separated in different UI components called **Midi Overlays** that provide more interesting features. Currently there are 3 overlays available:

- a Drag 'n Drop module 
- a Piano roll module (Midi Viewer)
- a (somewhat weird and experimental) Looper module that I've used to prototype the recording function.

All of them can be loaded into the [MidiOverlayPanel](/ui-components/floating-tiles/plugin/midioverlaypanel) floating tile so you can directly slap them on your UI with `"Index":0,1,2`. 

This list might get extended with additional modules over time, but if you can't wait, you can implement your own UI by connecting a ScriptPanel to the MIDI player and use the [MIDI Players Scripting API](/scripting/scripting-api/midiplayer) to create your own overlay.

## Scripting API

A [typed reference](/scripting/scripting-api/midiplayer) to a **MIDI Player** module gives you additional methods to:

1. Control the playback of the player with perfect sample accuracy
2. Perform MIDI processing on the files
3. Create custom UI Overlays by connecting it to a ScriptPanel which will be automatically updated on certain MIDI events.

### The MIDI Processing workflow

The MIDI Player module loads and plays MIDI files, but that is just the start: you can perform any kind of MIDI processing to the file after it was loaded using the same API calls that you would use for real time MIDI processing! The general workflow will always be the same three steps:

1. [Get a list of events](/scripting/scripting-api/midiplayer#geteventlist) from the MIDI file. It will return an array of `MessageHolder` objects that have the same API as the [Message](/scripting/scripting-api/message) class you know from MIDI processing live input
2. Iterate over the array and perform your processing. You can also delete events, but be aware that the `for(... in ...)` loop will not handle deletions well, so you need to resort to the `for(i=0; i < events.length; i++)` loop style.
3. [Flush the operation](/scripting/scripting-api/midiplayer#flushmessagelist) (writes back the array into the MIDI file). This operation is fully undoable so you can revert it back to the previous state if something went wrong.

A minimal example for this process would be:

```javascript
// fetch a typed reference
const var MIDIPlayer1 = Synth.getMidiPlayer("MIDI Player1");

// 1. get a list
const var list = MIDIPlayer1.getEventList();

// 2. Perform operations
for(e in list)
{
    if(e.isNoteOn())
        e.setVelocity(36);
}

// 3. Flush the processed list back to the MIDI file
MIDIPlayer1.flushMessageList(list);
```

Be aware that while the processing array contains [HiseEvents](/glossary/hise-event), the data will be stored as plain MIDI information when it's written back into the MIDI file, so you loose any special `HiseEvent` property like pitch fades, artificial state (plus the transpose amount will be merged with the note number).

The timestamp will use the current tempo and samplerate to be consistent with the rest of the MIDI processing in HISE. So if you have quarter notes at 44,1kHz and 120BPM, you will get these timestamps for the first bar:

```
0,
22050
44100
66150
```



