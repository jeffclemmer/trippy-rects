<style>
/* no style for this component */
</style>



<!-- add to window events -->
<svelte:window on:resize="{drawResize}"></svelte:window>

<!-- canvas to draw upon -->
<canvas id="canvas" bind:this="{canvas}" width="{width}" height="{height}" on:click="{ () => dispatch('click') }"></canvas>



<script>

// import event dispatch
import { createEventDispatcher } from 'svelte';
const dispatch = createEventDispatcher();

// external props
export let animating = true;
export let width = 0;
export let height = 0;
export let maxRects = 20;
export let lineWidth = 5;

// internal props
let canvas = null;
let ctx = null;

// runs when component is shown
import { onMount } from "svelte";
onMount(async () => {
  window.requestAnimationFrame(testFPS);
  
  if (canvas.getContext) {
    ctx = canvas.getContext("2d");
  }
  
  setDesiredRotation(0);
  setupRadians();
});


// start or stop animation
$:{
  // if animating switches to true, we need to trigger 
  // requestAnimationFrame to fire back up.
  if (animating == true) window.requestAnimationFrame(paint);
}


// fps testing from https://stackoverflow.com/questions/8279729/calculate-fps-in-canvas-using-requestanimationframe/64141620#64141620
let targetFPS = 15;
let fps = 1;
let times = [];
let start = 0;
let frameTick = 0;

// dummy frames let us skip frames we don't need to animate
let dummyFrame = 0, maxDummyFrames = 0;

// get a rough FPS estimate for the current browser session
function testFPS(timestamp) {
  // fps testing
  if (start == 0) start = timestamp;
  if (start < timestamp - 1200) {
    console.log("browser supported fps:", fps);
    
    maxDummyFrames = fps / targetFPS;
    console.log("target FPS:", targetFPS);
    
    // start animation
    window.requestAnimationFrame(paint);
    
    return;
  }
  
  while (times.length > 0 && times[0] <= timestamp - 1000) {
    times.shift();
  }
  timestamp = timestamp == undefined ? 0 : timestamp;
  times.push(timestamp);
  fps = times.length;
  window.requestAnimationFrame(testFPS);
}


function drawResize(e) {
  if (animating == false) {
    animating = true;
    dummyFrame = maxDummyFrames;
    window.requestAnimationFrame(paint);
  }
}


/* 
rects starts with one rect.  new rects are pushed on 
the stack and old ones get shifted off the bottom of the stack.
size is a multiplier of the width and height of the screen.
size must be between 1-19
rot is the rotation angle that the rect should be rotated.

this might not be the most memory efficient method, but it's the easiest
to implement.  also, i don't know what the underlying machine code is doing.
it might be pretty memory efficient.  however, in the future, i might
change this to a copy system where each item gets copied to the lower index
and then the new item gets copied into the highest index.

there is a concept of desired rotation that is randomly selected.
this allows the system to smoothly rotate the rect towards the desired
rotation.  once the desired rotation has been met, a new desired
rotation is randomly selected and the process starts again.
*/
let rects = [ {size: 1, rot: 0, r: 0, g: 180, b: 0, a: 1} ];
// let maxRects = 20;
let desiredRotation = 0;

// set the color of the next rect
let r = 0, g = 180, b = 0, a = 1;

// is the color ascending(2), descending(0), or staying the same(1)?
let rs = 2, gs = 1, bs = 1;

// for debugging purposes - sometimes things get out of hand
// set maxFrames to zero for unlimited frames
let curFrame = 0, maxFrames = 0;

function paint() {
  // if not animating or if not setup, return
  if (animating == false || fps == 1) return;
  
  // if not within a current draw frame, return
  // this skips frames so we get our desired lower frame rate
  if (dummyFrame < maxDummyFrames) {
    dummyFrame++;
    window.requestAnimationFrame(paint);
    return;
  }
  dummyFrame = 0;
  
  // clear screen
  ctx.clearRect(0, 0, width, height);
  
  // defined for later use
  let tx, ty;
  
  // find the center of the canvas
  let xcenter = width / 2;
  let ycenter = height / 2;

  // setup some colors and lines
  ctx.shadowColor = "#aaa";
  ctx.shadowBlur = lineWidth + 1;
  ctx.lineWidth = lineWidth;
  
  // loop through each rectangle to be drawn
  for (let i in rects) {
    
    let angle = rects[i].rot;
    
    // create a size multiplier
    let xmult = ( (width / maxRects) * rects[i].size) / 2;
    let ymult = ( (height / maxRects) * rects[i].size) / 2;
    
    // start drawing a rect with a path and setup the color
    ctx.beginPath();
    ctx.strokeStyle = `rgba(${rects[i].r},${rects[i].g},${rects[i].b},${rects[i].a})`;
    
    // top left of rect
    tx = 0 - xmult;
    ty = 0 - ymult;
    tx = tx * cos[angle] - ty * sin[angle];
    ty = ty * cos[angle] + tx * sin[angle];
    ctx.moveTo( tx + xcenter, ty + ycenter );
    
    // top right of rect
    tx = 0 + xmult;
    ty = 0 - ymult;
    tx = tx * cos[angle] - ty * sin[angle];
    ty = ty * cos[angle] + tx * sin[angle];
    ctx.lineTo( tx + xcenter, ty + ycenter );
    
    // bottom right of rect
    tx = 0 + xmult;
    ty = 0 + ymult;
    tx = tx * cos[angle] - ty * sin[angle];
    ty = ty * cos[angle] + tx * sin[angle];
    ctx.lineTo( tx + xcenter, ty + ycenter );
    
    // bottom left of rect
    tx = 0 - xmult;
    ty = 0 + ymult;
    tx = tx * cos[angle] - ty * sin[angle];
    ty = ty * cos[angle] + tx * sin[angle];
    ctx.lineTo( tx + xcenter, ty + ycenter );
    
    // top left again to complete rect
    tx = 0 - xmult;
    ty = 0 - ymult - (lineWidth / 2);
    tx = tx * cos[angle] - ty * sin[angle];
    ty = ty * cos[angle] + tx * sin[angle];
    ctx.lineTo( tx + xcenter, ty + ycenter );
    
    // draw shape
    ctx.stroke();
    
  }
  
  // apply pixelation
  ctx.beginPath();
  ctx.strokeStyle = "#0005";
  ctx.lineWidth = 1;
  ctx.shadowColor = "";
  ctx.shadowBlur = 0;
  
  for (let x = 2.5; x < width; x += 3) {
    ctx.moveTo(x, 0);
    ctx.lineTo(x, height);
  }
  
  for (let y = 2.5; y < height; y += 3) {
    ctx.moveTo(0, y);
    ctx.lineTo(width, y);
  }
  
  ctx.stroke();
  
  // do we need to shift a rect off?
  if (rects.length >= maxRects) {
    rects.shift();
  }
  
  // generate next rect
  
  // calc next rotation
  let lastRotation = rects[rects.length - 1].rot;
  
  // is desiredRotation less than lastRotation?
  if (desiredRotation < lastRotation) {
    lastRotation -= 2;
    
    // check if desiredRotation has been fullfilled
    if (lastRotation <= desiredRotation) {
      setDesiredRotation(lastRotation);
    }
  }
  
  // is desiredRotation greater than lastRotation?
  if (desiredRotation > lastRotation) {
    lastRotation += 2;
    
    // check if desiredRotation has been fullfilled
    if (lastRotation >= desiredRotation) {
      setDesiredRotation(lastRotation);
    }
  }
    
  rects.push({size: rects.length + 1, rot: lastRotation, r: r, g: g, b: b, a: 1});
  
  // fade out previous rects
  for (let i in rects) {
    rects[i].a -= 1 / maxRects;
    rects[i].size -= 1;
  }
  
  // muted rainbow algorithm
  
  // if going up...
  if (rs == 2) r += 30;
  if (gs == 2) g += 30;
  if (bs == 2) b += 30;
  
  // if going down...
  if (rs == 0) r -= 30;
  if (gs == 0) g -= 30;
  if (bs == 0) b -= 30;
  
  // saturate operation
  if (r < 0) r = 0;
  if (g < 0) g = 0;
  if (b < 0) b = 0;
  if (r > 180) r = 180;
  if (g > 180) g = 180;
  if (b > 180) b = 180;
  
  // if we're going up and we've reached max level
  if        (rs == 2 && r == 180) {
    rs = 1;
    gs = 0;
  } else if (gs == 0 && g == 0) {
    gs = 1;
    bs = 2;
  } else if (bs == 2 && b == 180) {
    bs = 1;
    rs = 0;
  } else if (rs == 0 && r == 0) {
    rs = 1;
    gs = 2;
  } else if (gs == 2 && g == 180) {
    gs = 1;
    rs = 2;
  }
  
  curFrame++;
  if (maxFrames == 0 || curFrame < maxFrames) {
    // console.log("rects:", rects);
    window.requestAnimationFrame(paint);
  } else {
    console.log("done");
  }
  
}


let spread = 400;
function setDesiredRotation(lastRotation) {
  
  // we use a "do" to make sure that desiredRotation isn't zero
  do {
    desiredRotation = Math.round( (Math.random() * spread) - (spread / 2) );
    // console.log("new desiredRotation:", desiredRotation);
  } while (desiredRotation == 0 && desiredRotation == lastRotation);
}

/* 
we want to turn a traditional angle into it's final cosine or sine 
component.  we use a lookup table, since our needs are pretty simple.
it's fast to use a lookup table, as opposed to having to calculate sin
and cos each time.

{a: angle, f: cos or sin}
for cos or sin, the angle = cos or sin calculation output
we use an associative array to compose this.
cos[angle] = cos output

*/
let cos = [];
let sin = [];

function setupRadians() {
  let start = 0 - spread / 2;
  let end = spread - spread / 2;
  
  for (let angle = start; angle <= end; angle++) {
    let radian = angle * Math.PI / 180;
    cos[angle] = Math.cos(radian);
    sin[angle] = Math.sin(radian);
  }
}


</script>