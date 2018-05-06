---
layout: post
title: iOS App Summary
---


# Projects Update
I've been learning some iOS development through building different apps. So I'll summarize current status, lessons learned, challenges, and direction forward.

# PianoPal
A piano tool for learning and visualizing scales and chords on a scrollable and listenable keyboard.

Source: <https://github.com/jescriba/pianopal> 

### Background
I wanted to work on another tool targeted toward beginner piano musicians - namely _myself_ :p When I'm on the go and want to visualize and hear different chords or scales. Ya know, develop an intuition for how a chord/scale name translates to a _sound_ and _feeling_.

 **Video of usage**
 
 <iframe width="532" height="313" src="https://youtube.com/embed/T1oxFbqeOm0" frameborder="0" allowfullscreen></iframe>

### Basic Principles

**Setting up infinite scroll UI**

For infinite scrolling on the keyboard - I implemented a scroll view with the keys in an octave and set the scroll view's `contentSizeWidth` to be 3x the screen width. The keyboard has an octave on screen (center octave), an octave left off screen, and another octave right off the screen. When the user scrolls to the end of either the left or right octave, the scroll view is recentered creating the illusion of an infinite scroll. Here's the [snippet](https://github.com/jescriba/pianopal/blob/332e413ffe55fb75159807397ede8859e266518a/pianopal/UI/PianoView.swift#L81) showing how this works currently in the `UIScrollViewDelegate`:

```swift
func scrollViewDidScroll(_ scrollView: UIScrollView) {
    var scrollDirection: ScrollDirection?
    if (lastContentOffset != nil) {
        if scrollView.contentOffset.x > lastContentOffset {
            scrollDirection = ScrollDirection.rightToLeft
        } else {
            scrollDirection = ScrollDirection.leftToRight
        }
    }
    lastContentOffset = scrollView.contentOffset.x
    switch scrollView.contentOffset.x {
    case scrollView.frame.width * 2, 0:
        scrollView.setContentOffset(CGPoint(x: scrollView.frame.width, y: 0), animated: false)
        updateOctave(scrollDirection)
        break
    default:
        break
    }
}
```

**Representation of notes, chords, and scales**

The [notes](https://github.com/jescriba/pianopal/blob/332e413ffe55fb75159807397ede8859e266518a/pianopal/Models/Note.swift#L11) are represented with `enum Note : Int` with a few helper methods for getting human-readable descriptions. It's useful having note inherit from `Int` since it allows for some basic math in calculating chords/scales. The [chords](https://github.com/jescriba/pianopal/blob/332e413ffe55fb75159807397ede8859e266518a/pianopal/Models/Chord.swift#L11) have an array of notes and a `ChordType`. The `ChordType` is another enum with the high level chord name i.e. Diminished, Augmented, etc... and a `ChordFormula` method:

```swift
func chordFormula() -> [Int] {
    switch self {
    case .Major:
        return [4, 7]
    case .Minor:
        return [3, 7]
    case .Diminished:
        return [3, 6]
    case .Augmented:
        return [4, 8]
    case .Sus2:
        return [2, 7]
    case .Sus4:
        return [5, 7]
    case .MajorSeventh:
        return [4, 7, 11]
    case .MinorSeventh:
        return [3, 7, 10]
    case .DominantSeventh:
        return [4, 7, 10]
    case .DiminishedSeventh:
        return [3, 6, 9]
    case .HalfDiminishedSeventh:
        return [3, 6, 10]
    }
}
```

Which gives you the intervals for each successive note with respect to the root note. For example, starting with a Cmaj you take the C then 4 semitones up you get the E then the G is 7 semitones up from the root. So adding new chords to the app is just a matter of adding another chord type name and formula. The scales are handled similarly with a [ScaleType](https://github.com/jescriba/pianopal/blob/332e413ffe55fb75159807397ede8859e266518a/pianopal/Models/ScaleType.swift#L11).

Both the `Chord` and `Scale` types are `NSCoding` compliant allowing these values to be easily stored in `UserDefaults`. This allows users to save chord and scale progressions.

For details on chord identification from tapped notes see the [ChordIdentifier](https://github.com/jescriba/pianopal/blob/332e413ffe55fb75159807397ede8859e266518a/pianopal/Helpers/ChordIdentifier.swift#L11).

**How the audio playback works**

The `audio/` folder of the source contains numerous notes recorded as mp3s i.e. A1.mp3, B1.mp3, etc...  These notes were recorded on my digital piano with the help of midi for duration and volume consistency. The `NoteOctave` model has a function that returns the appropriate url for the audio file in the main bundle that is used by the `AudioEngine` to play the file. For example playing a scale:

```swift
let file = try? AVAudioFile(forReading: note.url() as URL)
let buffer = AVAudioPCMBuffer(pcmFormat: file!.processingFormat, frameCapacity: AVAudioFrameCount(file!.length))
_ = try? file?.read(into: buffer)
var completionHandler: AVAudioNodeCompletionHandler?
if index == notes.count - 1 {
    completionHandler = {
        self.delegate?.didFinishPlayingNotes(notes)
        self.delegate?.didFinishPlaying()
    }
} else {
    completionHandler = {
        self.delegate?.didFinishPlayingNotes([note])
        self.delegate?.didStartPlayingNotes([notes[index + 1]])
    }
}
scalePlayer?.scheduleBuffer(buffer,
                            completionHandler: completionHandler)
scalePlayer?.play()
if index == 0 {
    delegate?.didStartPlayingNotes([note])
}
```

Here the delegate is used for updating the UI - highlighting the key as the note is being played.

### Pitfalls and Improvements

* Didn't utilize much Auto-Layout even for basic views which would have saved me from some headache in setting up the UI in code.

Moving forward, I'd love to update the 'play modes' so scales could be played in arpeggio and chord progressions can be played for you in sequence or arpeggiated.

# Synthia
A rudimentary syntheizer and sequencer. 

Source: <https://github.com/jescriba/synthia>

### Background
For my first app, I wanted to get some experience building a creative tool using the [AVAudioEngine](https://developer.apple.com/reference/avfoundation/avaudioengine) as a fun way of learning the basics of iOS development.

 **Note**: This app predates Swift 3 so some of the syntax might look off but you get the idea - if it bugs you feel free to open up a PR. ;)
 
 **Video of usage**
 
 <iframe width="532" height="313" src="https://www.youtube.com/embed/oTBtwi3UoMk" frameborder="0" allowfullscreen></iframe>


### Basic Principles

**The Sequencer implementation**

The sequencer plays audio samples from <https://www.freesound.org>. With the bpm of the sequencer being calculated and passed into an `NSTimer`. (See pitfalls for more details)

**The Keyboard implementation**

The keyboard, more excitingly, plays audio by playing parallel 'voices' or oscillators using a [circular buffer](https://en.wikipedia.org/wiki/Circular_buffer). The notes get mapped to a frequency in a dictionary then the value of the wave is calculated and placed in the audio buffer. [Here](https://github.com/jescriba/synthia/blob/3aa5372ec47534704f488daceebb8b5de33fe3a2/Synthia/Voice.swift#L53) is starting a note for a voice:

```swift 
func startNote() {
    let unitVelocity = 2.0 * M_PI / audioFormat.sampleRate
    dispatch_async(audioQueue) {
        var sampleTime = 0
        var previousKey = self.key
        while (true) {
            if self.key != previousKey {
                self.computeScaleBaseFrequenciesForKey(self.key!)
                previousKey = self.key
            }
            dispatch_semaphore_wait(self.audioSempahore, DISPATCH_TIME_FOREVER)
            let targetFrequency = Float32(Double(self.octave!) * self.scaleBaseFrequencies![self.noteIndex!])
            let audioBuffer = self.audioBuffers[self.bufferIndex]
            for i in 0...Int(self.samplesPerBuffer - 1) {
                let carrierVelocity = targetFrequency * Float32(unitVelocity)
                var sample = Float32(0)
                var triangleSample = Float32(0)
                var squareSample = Float32(0)
                var sineSample = Float32(0)
                
                // WaveTypes
                triangleSample = Float32(Float(2.0 / M_PI) * asin(sin(Float(M_PI) * Float(carrierVelocity) * Float(sampleTime))))
                
                sineSample = 0
                
                var intermediateVal = sinf(Float(carrierVelocity) * Float(sampleTime));
                // sgn function
                if (intermediateVal < 0) {
                    intermediateVal = -1;
                } else if (intermediateVal > 0) {
                    intermediateVal = 1;
                }
                squareSample = 1 / 2 * intermediateVal;
                
                sample = self.squareWaveRatio * squareSample + self.triangleWaveRatio * triangleSample + self.sineWaveRatio * sineSample
                
                
                audioBuffer.floatChannelData[0][i] = sample
                audioBuffer.floatChannelData[1][i] = sample
                sampleTime += 1
            }
            audioBuffer.frameLength = self.samplesPerBuffer
            self.oscNode.scheduleBuffer(audioBuffer, completionHandler: { () -> Void in
                dispatch_semaphore_signal(self.audioSempahore)
                return
            })
            self.bufferIndex = (self.bufferIndex + 1) % self.audioBuffers.count
        }
    }
}
```
    
**Circular buffer, audio buffer, `audioSempahore`, `dispatch_sempahore_wait`, wuhhh?**

A little more detail in the audio format and setup:

```swift
let audioFormat = AVAudioFormat(standardFormatWithSampleRate: 44100.0, channels: 2)
var audioBuffers = [AVAudioPCMBuffer]()
let audioQueue = dispatch_queue_create("SynthQueue", DISPATCH_QUEUE_SERIAL)
let audioSempahore = dispatch_semaphore_create(2)
let samplesPerBuffer = AVAudioFrameCount(1024)   
```
    
The `PCMBuffer` represents a sampled wave form ([PCM](https://en.wikipedia.org/wiki/Pulse-code_modulation)) placed into a buffer. There's a queue for the currently playing audio buffer and the one to be scheduled after. You can think of this as playing a 'snippet' of the waveform, and you'll want to play these 'snippets' sequentially, without delay, to recreate the waveform you hear. Once the playing audio buffer finishes it is then calculated and scheduled again and so on - hence the 'loop' or 'circular' buffer. In code, the `AVAudioPCMBuffer` array of size 2 represents this loop - since after the 1st buffer is played the 0th buffer can be recalculated and scheduled after. This is why the bufferIndex is recalculated after scheduling the buffer with: `self.bufferIndex = (self.bufferIndex + 1) % self.audioBuffers.count`

 The playing buffer has a completion handler that signals the sempahore - allowing for passage through the `dispatch_semaphore_wait(self.audioSempahore, DISPATCH_TIME_FOREVER)
` statement thus calculating and filling the buffer to be played next. The key here is that a buffer needs to always be scheduled before the currently playing buffer finishes - otherwise you'll hear hiccups in the audio. 
    
**Where the heck did these `triangleSample` and `squareSample`s come from?**

Wolfram is your friend - with great resources describing analytic representations of different wave forms i.e. [TriangleWave](http://mathworld.wolfram.com/TriangleWave.html).

**How do the effects like EQ, delay, reverb, etc... work?** 

`AVFoundation` provides a lot of these tools _for you_ woohoo. In my [AudioEngine](https://github.com/jescriba/synthia/blob/3aa5372ec47534704f488daceebb8b5de33fe3a2/Synthia/AudioEngine.swift) implementation you can see how to hook up various effects to the `AVAudioPlayerNode` like the `AVAudioUnitDelay` which has intuitive high-level parameters _wetDryMix_. [Here](https://github.com/jescriba/synthia/blob/3aa5372ec47534704f488daceebb8b5de33fe3a2/Synthia/AudioEngine.swift#L38) is the initialization for my `AudioEngine` which relies on the `AVAudioEngine`:

```swift
init (numberOfVoices: Int, withKey: String, withOctave: Int) {
    playbackFileUrl = nil
    engine.attachNode(dryMasterMixerNode)
    for _ in 0...(numberOfVoices - 1) {
        let voice = Voice(withKey: withKey, withOctave: withOctave)
        voiceArray.append(voice)
        engine.attachNode(voice.oscNode)
        engine.connect(voice.oscNode, to: dryMasterMixerNode, format: audioFormat)
    }
    delayNode = AVAudioUnitDelay()
    delayNode!.bypass = false
    delayNode!.delayTime = 0.68
    delayNode!.wetDryMix = 10
    reverbNode = AVAudioUnitReverb()
    reverbNode!.bypass = false
    reverbNode!.wetDryMix = 20
    eqNode = AVAudioUnitEQ(numberOfBands: 1)
    eqNode!.bands.first!.filterType = AVAudioUnitEQFilterType.LowPass
    eqNode!.bands.first!.frequency = 800
    eqNode!.bands.first!.bypass = false
    eqNode!.globalGain = 0
    eqNode!.bypass = false
    distortionNode = AVAudioUnitDistortion()
    distortionNode!.bypass = false
    distortionNode!.wetDryMix = 0
    engine.attachNode(delayNode!)
    engine.attachNode(reverbNode!)
    engine.attachNode(eqNode!)
    engine.attachNode(distortionNode!)
    engine.attachNode(playbackFilePlayer)
    engine.connect(dryMasterMixerNode, to: delayNode!, format: audioFormat)
    engine.connect(delayNode!, to: reverbNode!, format: audioFormat)
    engine.connect(reverbNode!, to: eqNode!, format: audioFormat)
    engine.connect(eqNode!, to: distortionNode!, format: audioFormat)
    engine.connect(distortionNode!, to: engine.mainMixerNode, format: audioFormat)
    engine.connect(playbackFilePlayer, to: engine.mainMixerNode, format: audioFormat)
    
    do {
        try engine.start()
    } catch {
        print("Engine Crashed")
    }
    
    let audioSession = AVAudioSession.sharedInstance()
    do {
        try audioSession.setActive(true)
        try audioSession.setCategory("AVAudioSessionCategoryPlayback")

    } catch  {
        print("audio session crash")
    }
}
```    
    
Notice how you essentially attach nodes to an engine and then connect them accordingly - similar to a real 'fx' chain like guitar pedals or rack units.

**Is the timbre changing???**

One neat aspect that I felt utilized the screen space is the fact that the position ('y'- vertical position) of your finger on the key changes the timbre of the oscillator by changing the 'wave ratio' - the amount of squareness in the wave - giving a more buzzy tone further down you go. To see how I implemented this gesture location tracking the brute force way, check out the [PadHandler](https://github.com/jescriba/synthia/blob/3aa5372ec47534704f488daceebb8b5de33fe3a2/Synthia/PadHandler.swift). (This implementation can be drastically improved with gesture recognizers mentioned in the Pitfalls section). This feature could be expanded to utilize _force touch_ [APIs](https://developer.apple.com/library/content/documentation/UserExperience/Conceptual/Adopting3DTouchOniPhone/3DTouchAPIs.html) on the newer iPhones to have parameter affect the timbre or fx of the oscillator. I didn't feel compelled to do so since I'm personally hanging on to my cracked iPhone 6 and 5. 

### Pitfalls and Improvements

* Overall lack of [gesture recognizers](https://developer.apple.com/reference/uikit/uigesturerecognizer?changes=latest_minor). Using gesture recognizers would have made my life a lot easier for handling key presses and sequencer events. Currently the targets are manually added and connected to a selector with `addTarget`.
 _One improved approach_ would be to have the keyboard as a custom view in a separate xib with all the appropriate tap recognizers on it for event handling and triggering the audio oscillators. _Another improved approach_ could be to use a collection view for both the keyboard and sequencer.
* A lot of hard coded or calculated UI code. Basically, the entire UI is based on ratios off the screen bounds, `UIScreen.mainScreen().bounds`. Leaving for a lot of calculated code like the key pad frame. For example:

```swift
let widthOfPad = Int(ceil(Double(Int(screenWidth) - 2 * numberOfPads + 2) / Double(numberOfPads)))
let heightOfPad = Int(screenHeight - 42)
```

*  The sequencer doesn't use a collection view. Currently it's a calculated grid of `UIButton`s with the appropriate target attached. [For example](https://github.com/jescriba/synthia/blob/3aa5372ec47534704f488daceebb8b5de33fe3a2/Synthia/SequencerViewController.swift#L110):

```swift
func calculateGrid() {
    drumButtons.removeAll()
    let remainingHeight = screenHeight - toolbarView!.frame.height
    let drumBtnHeight = remainingHeight / CGFloat(numberOfRows)
    let drumBtnWidth = screenWidth / CGFloat(numberOfColumns)
    for row in 0...(numberOfRows - 1) {
        var columnArray = [UIButton]()
        for col in 0...(numberOfColumns - 1) {
            let x = CGFloat(col) * drumBtnWidth
            let y = CGFloat(row) * drumBtnHeight + toolbarView!.frame.height
            let btn = UIButton(frame: CGRect(x: x, y: y, width: drumBtnWidth, height: drumBtnHeight))
            btn.backgroundColor = padOffColor
            btn.layer.borderWidth = 0.5
            btn.layer.borderColor = btnBorderColor.CGColor
            btn.tag = row * numberOfColumns + col
            btn.addTarget(self, action: #selector(SequencerViewController.toggleDrumBtn(_:)), forControlEvents: UIControlEvents.TouchUpInside)
            view.addSubview(btn)
            columnArray.append(btn)
        }
        drumButtons.append(columnArray)
    }	
}
```

The keyboard works [similiarly](https://github.com/jescriba/synthia/blob/3aa5372ec47534704f488daceebb8b5de33fe3a2/Synthia/SynthViewController.swift#L83).
    
*  The sequencer doesn't properly sequence audio samples. Currently all the samples are triggered when the UI for the row updates via an `NSTimer`. [For example](https://github.com/jescriba/synthia/blob/3aa5372ec47534704f488daceebb8b5de33fe3a2/Synthia/SequencerViewController.swift#L219):

```swift
func playStep() {
    if isPlaying {
        playIndex = playIndex % numberOfColumns
        let indicesToPlay = getIndicesToPlayAndHighlightColumn(playIndex)
        audioEngine!.triggerDrumSamplesForIndices(indicesToPlay)
        playIndex += 1
        let timeInterval = NSTimeInterval(Float(240) / Float(numberOfColumns * bpm))
        NSTimer.scheduledTimerWithTimeInterval(timeInterval, target: self, selector: #selector(SequencerViewController.playStep), userInfo: nil, repeats: false)
    }
}
```

The AudioEngine then plays the indices of the drum grid by scheduling the audio files to play in the array of `DrumSample`s. [Here](https://github.com/jescriba/synthia/blob/3aa5372ec47534704f488daceebb8b5de33fe3a2/Synthia/DrumSample.swift#L39):

```swift
func trigger() {
	if !hasCompletedPlayback {
	    playerNode.stop()
	}
	hasCompletedPlayback = false
	playerNode.scheduleFile(sampleFile!, atTime: nil, completionHandler: {
	    self.hasCompletedPlayback = true
	})
	playerNode.play()
}
```

Note this **is not** a good approach and causes problems when the sequencer starts lagging for a sample to complete playing and the audio events get 'backed' up. The correct approach probably involves actually using a properly calculated `AVAudioTime` ([docs](https://developer.apple.com/reference/avfoundation/avaudiotime)) to schedule the files with any previously playing samples in a row being terminated before completion, if appropriate.

Maybe if I get around to picking up this project again I'll refactor and fix these notable issues. Or just start a project from scratch.

# Moving Forward
After learning more about app development especially using Auto-Layout through sample apps with [CodePath](https://github.com/codepath), I've started looking forward to making a new app utilizing some of these learned lessons as well as an increased familarity with Xcode.

### Tarty

An Art educational tool using the [Artsy API](https://developers.artsy.net/). Uses Auto-Layout, gesture recognizers, delegates, and collection views as improvements on older patterns mentioned above.

**Currently in preliminary development.**

Source: <https://github.com/jescriba/tarty>
