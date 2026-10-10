# nnls-chroma

Vamp plugins for chroma features and chord recognition from audio, by Matthias Mauch and Chris Cannam (Queen Mary, University of London). The library holds three plugins: NNLS Chroma, Chordino and Tuning. This fork adds a build for Apple Silicon.

<img src="chordino.png">

## Requirements

Xcode command-line tools, Boost headers (`brew install boost`) and the Vamp plugin SDK. To call the plugins from Python: `pip install vamp`.

## Build

Clone the Vamp plugin SDK next to this directory and build its static library, then the plugin:

```
brew install boost
git clone https://github.com/vamp-plugins/vamp-plugin-sdk
make -C vamp-plugin-sdk -f otherbuilds/Makefile.osx sdkstatic ARCHFLAGS="-mmacosx-version-min=11.0 -arch arm64 -stdlib=libc++"
make -C nnls-chroma -f Makefile.osx
```

This writes `nnls-chroma.dylib`. `VAMP_SDK_DIR` and `BOOST_ROOT` can be set on the `make` command line. `Makefile.linux` and `Makefile.mingw` build for Linux and Windows.

## Usage

Put `nnls-chroma.dylib` and `nnls-chroma.cat` in a folder and point `VAMP_PATH` at it. Any Vamp host can load them (Sonic Visualiser, Sonic Annotator, the `vamp` Python package).

```python
import librosa, vamp
y, sr = librosa.load("song.mp3", sr=44100, mono=True)
chords = vamp.collect(y, sr, "nnls-chroma:chordino", output="simplechord")["list"]
for c in chords:
    print(c["timestamp"], c["label"])
```

Each entry marks a chord change: the time it starts and the label, like `Am`, `G/B` or `Emaj7`. `N` is no chord. A chord lasts until the next entry.

Parameters go in as `parameters={"boostn": 0.2}`.

## Plugins

### NNLS Chroma (`nnls-chroma:nnls-chroma`)

Turns the spectrum into a log-frequency spectrum with three bins per semitone, corrects for tuning and whitens it. Then it estimates which notes are playing (NNLS approximate transcription, a dictionary of harmonic notes with decaying overtones) and folds the notes into 12-bin chromagrams.

| Output | Content |
|---|---|
| `logfreqspec` | Log-frequency spectrum, 3 bins per semitone |
| `tunedlogfreqspec` | The same after tuning, background subtraction and whitening |
| `semitonespectrum` | One bin per semitone. With NNLS on, the note activations |
| `chroma` | 12-bin chromagram weighted to the mid range |
| `basschroma` | 12-bin chromagram of the bass range |
| `bothchroma` | Bass and mid chromagrams, 24 bins |

### Chordino (`nnls-chroma:chordino`)

Chord recognition on top of NNLS Chroma. Scores each frame against chord profiles and smooths the result with an HMM (Viterbi). The profiles come from `chord.dict` if one is in the plugin folder, otherwise from a built-in set.

| Output | Content |
|---|---|
| `simplechord` | Chord changes: time and label |
| `chordnotes` | The notes of each chord, bass note included, as MIDI-like notes |
| `harmonicchange` | How likely a chord change is at each frame |
| `loglikelihood` | Log-likelihood of the chord estimate |

### Tuning (`nnls-chroma:tuning`)

Estimates the concert pitch in Hz, the same way the other two plugins do.

| Output | Content |
|---|---|
| `tuning` | One value for the whole piece |
| `localtuning` | A running estimate per frame |

## Parameters

Defaults are the settings from Matthias Mauch's 2010 MIREX submissions.

| Parameter | Plugins | Range | Default | Meaning |
|---|---|---|---|---|
| `useNNLS` | Chroma, Chordino | 0 or 1 | 1 | 1: NNLS transcription. 0: map the spectrum straight to semitones |
| `rollon` | all | 0 to 5 (%) | 0 | Bass noise threshold: bins below this share of the cumulative energy are set to 0 |
| `tuningmode` | Chroma, Chordino | 0 or 1 | 0 | 0: one tuning for the whole piece. 1: local tuning, for audio whose tuning drifts |
| `whitening` | Chroma, Chordino | 0 to 1 | 1 | How much the log-frequency spectrum is whitened. 0 is none |
| `s` | Chroma, Chordino | 0.5 to 0.9 | 0.7 | Note shape in the NNLS dictionary: harmonic i has amplitude s^(i-1) |
| `chromanormalize` | Chroma | 0 to 3 | 0 | Divide each chroma frame by its norm: 0 none, 1 maximum, 2 L1, 3 L2. Changes `bothchroma` relative to the 12-bin outputs |
| `boostn` | Chordino | 0 to 1 | 0.1 | Makes `N` (no chord) more likely. Above the default, real chords quickly turn into `N` |
| `usehartesyntax` | Chordino | 0 or 1 | 0 | 1: labels in Harte syntax (`A:min`) instead of `Am` |

## Files

| File | Purpose |
|---|---|
| `NNLSBase.cpp`, `NNLSChroma.cpp`, `Chordino.cpp`, `Tuning.cpp` | The three plugins and their shared base |
| `chromamethods.cpp`, `nnls.c`, `viterbi.cpp` | Note dictionary, NNLS solver, Viterbi decoding |
| `plugins.cpp`, `vamp-plugin.list`, `vamp-plugin.map` | Plugin entry point and exported symbols |
| `nnls-chroma.cat`, `nnls-chroma.n3` | Plugin category and RDF description for Vamp hosts |
| `chordextract.cpp` | Command-line chord extractor |
| `regression/` | Regression test: `make -f Makefile.osx test` |

## Credits

Matthias Mauch and Chris Cannam, Queen Mary, University of London. The authors ask that public or commercial use mention them and Queen Mary, University of London in user-visible documentation, separately from the license.

For academic use, cite: Matthias Mauch and Simon Dixon, [Approximate Note Transcription for the Improved Identification of Difficult Chords](http://schall-und-mauch.de/artificialmusicality/?p=89), Proceedings of the 11th International Society for Music Information Retrieval Conference (ISMIR 2010), 2010. BibTeX in `CITATION`.

## License

This program is free software: you can redistribute it and/or modify
it under the terms of the GNU General Public License as published by
the Free Software Foundation, either version 2 of the License, or (at
your option) any later version.

This program is distributed in the hope that it will be useful, but
WITHOUT ANY WARRANTY; without even the implied warranty of
MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the GNU
General Public License for more details.

You should have received a copy of the GNU General Public License
along with this program.  If not, see <http://www.gnu.org/licenses/>.

Changes in this fork by nashaad, 2026.
