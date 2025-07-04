<script lang="ts">
  import data from "./song.json";
  import midimap from "./midimapping.json";
  import Vex from "vexflow";
  import {
    Stave,
    StaveNote,
    StaveConnector,
    Dot,
    Factory,
    Formatter,
    Voice,
    Renderer,
    Beam,
  } from "vexflow";
  import { onMount } from "svelte";

  interface Note {
    eventType: string;
    delta: number;
    duration: number;
    midinote: number;
    velocity: number;
  };

  const durationToTicks: Record<number, string> = {
    "1920": "w",
    "1440": "hd",
    "960": "h",
    "720": "qd",
    "480": "q",
    "360": "8d",
    "320": "8d",
    "240": "8",
    "180": "16d",
    "160": "16d",
    "120": "16",
    "90": "32d",
    "60": "32",
    "30": "64",
  };

  let containerRef;
  // Create the notes and project vars
  const notes: StaveNote[][] = [];
  let noteBuffer: StaveNote[] = [];
  const BPM = data.bpm;
  const timeSig = data.timeSig;
  let STAVE_WIDTH = $state(250);
  let STAVE_HEIGHT = $state(300);
  let staveIter = 0;
  let lineIter = 0;
  // Create a mapping from MIDI note number to note name
  const midiNoteToName = new Map(
    midimap.map((item) => [
      item["MIDI note number"],
      item["Note names (English)"],
    ]),
  );
  

  function convertNoteToVexFlow(noteName: string | undefined): string {
    // Handle cases like "F#7/Gb7" - just take the first one
    if (noteName === undefined) {
      console.warn(`Could not parse note name: ${noteName}`);
      return ""; // Return an empty string or handle error appropriately
    }
    const primaryNote = noteName.split("/")[0];

    // Extract note letter and octave
    const match = primaryNote.match(/^([A-G])(#|b)?(\d+)$/);
    if (!match) {
      console.warn(`Could not parse note name: ${noteName}`);
      return ""; // Return an empty string or handle error appropriately
    }

    const noteLetter = match[1].toLowerCase();
    const accidental = match[2] || "";
    const octave = (parseInt(match[3])+1);  ///// MANUAL FIX HERE, NEED TO CHANGE, just for testing rn.
    return `${noteLetter}${accidental}/${octave}`;
  }
  function dotted(staveNote: StaveNote, noteIndex = -1) {
      if (noteIndex < 0) {
        Dot.buildAndAttach([staveNote], {
          all: true,
        });
      } else {
        Dot.buildAndAttach([staveNote], {
          index: noteIndex,
        });
      }
      return staveNote;
  }

  function setupVexFlow() {
    // const vf = new Factory({
    //   renderer: { elementId: "containerRef", width: 1000, height: 400 },
    // });
    // const renderer = new Renderer("containerRef", Renderer.Backends.SVG);
    // // Configure the rendering context.
    // renderer.resize(1000, 1000);
    // let context = renderer.getContext();
    // // Create a stave of width 400 at position 10, 40 on the canvas.
    // const prevStaveMeasure = new Stave(0, 0, STAVE_WIDTH);

    // // Add a clef and time signature.
    // prevStaveMeasure.addClef("treble").addTimeSignature(timeSig);

    // // Connect it to the rendering context and draw!
    // prevStaveMeasure.setContext(context).draw();
  }
  function drawNotes(
      context: any,
      firstStave: Stave,
      all_notes: StaveNote[][],
      staveIter: number,
      lineIter: number,
    ) {
      for (let i = 0; i < all_notes.length; i++) {
        if (i === 0) {
          const bar = [...all_notes[i]]
          var beams = Beam.generateBeams(bar);
          Formatter.FormatAndDraw(context, firstStave, bar, { autoBeam: true, alignRests: true});
          staveIter++;
        } else {
        // Create a new stave for each chunk
        if (staveIter === 4) {
          staveIter = 0;
          lineIter++;
        }
        const newStaveMeasure = new Stave(
          staveIter * STAVE_WIDTH,
          lineIter * STAVE_HEIGHT,
          STAVE_WIDTH,
        );
        newStaveMeasure.setContext(context).draw();
        const bar = [...all_notes[i]]
        var beams = Beam.generateBeams(bar);
        
        Formatter.FormatAndDraw(context, newStaveMeasure, bar, { autoBeam: true, alignRests: true});
        staveIter++;
        }
      }
  }
  function gatherNotes(noteBuffer: StaveNote[], noteArray: StaveNote[][] = []) {
    let barDurBuffer = 0;
    for (let i = 0; i < data.tracks[0].length; i++) {
      const note: Note = data.tracks[0][i];
      const noteName: string | undefined = midiNoteToName.get(note.midinote.toString());
      const vexFlowNote = convertNoteToVexFlow(noteName);
      const dur = durationToTicks[note.duration];
      if (dur.includes("d")) {
        noteBuffer.push(
          dotted(new StaveNote({ keys: [vexFlowNote], duration: dur })),
        );
        barDurBuffer += note.duration;
        // console.log(`Note: ${vexFlowNote}, Duration: ${dur}, noteBuffer:`);
      } else {
        noteBuffer.push(new StaveNote({ keys: [vexFlowNote], duration: dur }));
        barDurBuffer += note.duration;
        // console.log(`Note: ${vexFlowNote}, Duration: ${dur}, noteBuffer:`);
        
      }
      if (barDurBuffer >= 1900) {
        noteArray.push([...noteBuffer]);
        noteBuffer.length = 0;
        barDurBuffer = 0;
      }
    }
    // After the loop, if there are any remaining notes in the buffer (less than 4)
    // add them to the noteArray as well.
    if (barDurBuffer <= 1920) {
        noteArray.push([...noteBuffer]);
        console.dir([...noteArray])
        noteBuffer.length = 0; // Clear the buffer
    }
  }
  onMount(() => {
    const vf = new Factory({
      renderer: { elementId: "containerRef", width: 0, height: 0 },
    });
    const renderer = new Renderer("containerRef", Renderer.Backends.SVG);
    // Configure the rendering context.
    renderer.resize(1200, 1200);
    let context = renderer.getContext();
    // Create a stave of width 400 at position 10, 40 on the canvas.
    const firstStaveMeasure = new Stave(0, 0, STAVE_WIDTH);

    // Add a clef and time signature.
    firstStaveMeasure.addClef("treble").addTimeSignature(timeSig);

    // Connect it to the rendering context and draw!
    firstStaveMeasure.setContext(context).draw();
    setupVexFlow();
    gatherNotes(noteBuffer, notes);
    drawNotes(context, firstStaveMeasure, [...notes], staveIter, lineIter);
  });

</script>



<main>
  <div id="containerRef"></div>
</main>


