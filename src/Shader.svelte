<svelte:window bind:innerWidth={innerWidth} bind:innerHeight={innerHeight}/>

<script>

	import { onMount } from 'svelte';
	import { createProgramFromSources } from './ShaderUtils.svelte';

	export let scale;

	let canvas;
	let canvasWidth, canvasHeight;
	let clientWidth, clientHeight;

	let innerWidth, innerHeight;

	let m = { x: 0, y: 0 };
	let time = 0;

	function handleMousemove(event) {

		let s = (scale * 100);

		canvasWidth = s + "%";
		canvasHeight = s + "%";

		m.x = (event.clientX / clientWidth) * s;
		m.y = (event.clientY / clientHeight) * s;
	}

	onMount(() => {
		const gl = canvas.getContext('webgl');
		let frame;

		if (!gl) {
			return; // Fallback options for no webgl support
		}

		// Defaul Vertex Shader
		const vs = `
			// an attribute will receive data from a buffer
			attribute vec4 a_position;

			// all shaders have a main function
			void main() {

			// gl_Position is a special variable a vertex shader
			// is responsible for setting
			gl_Position = a_position;
			}
		`;

		// Defaul Fragment Shader
		const fs = `
			precision highp float;

			uniform vec2 u_resolution;
			uniform vec2 u_mouse;
			uniform float u_time;

			void main() {
			// gl_FragColor is a special variable a fragment shader
			// is responsible for setting

			gl_FragColor = vec4(fract((gl_FragCoord.xy - u_mouse) / u_resolution), fract(u_time), 1);
			}
		`;

		// setup GLSL program
		const program = createProgramFromSources(gl, [vs, fs]);

		// look up where the vertex data needs to go.
		const positionAttributeLocation = gl.getAttribLocation(program, "a_position");

		// look up uniform locations
		const resolutionLocation = gl.getUniformLocation(program, "u_resolution");
		const mouseLocation = gl.getUniformLocation(program, "u_mouse");
		const timeLocation = gl.getUniformLocation(program, "u_time");

		// Create a buffer to put three 2d clip space points in
		const positionBuffer = gl.createBuffer();

		// Bind it to ARRAY_BUFFER (think of it as ARRAY_BUFFER = positionBuffer)
		gl.bindBuffer(gl.ARRAY_BUFFER, positionBuffer);

		// fill it with a 2 triangles that cover clipspace
		gl.bufferData(gl.ARRAY_BUFFER, new Float32Array([
			-1, -1,  // first triangle
			1, -1,
			-1,  1,
			-1,  1,  // second triangle
			1, -1,
			1,  1,
		]), gl.STATIC_DRAW);


		(function loop() {
			frame = requestAnimationFrame(loop);
			time += .01;

			// Tell WebGL how to convert from clip space to pixels
			gl.viewport(0, 0, gl.canvas.width, gl.canvas.height);

			// Tell it to use our program (pair of shaders)
			gl.useProgram(program);

			// Turn on the attribute
			gl.enableVertexAttribArray(positionAttributeLocation);

			// Bind the position buffer.
			gl.bindBuffer(gl.ARRAY_BUFFER, positionBuffer);

			// Tell the attribute how to get data out of positionBuffer (ARRAY_BUFFER)
			gl.vertexAttribPointer(
				positionAttributeLocation,
				2,          // 2 components per iteration
				gl.FLOAT,   // the data is 32bit floats
				false,      // don't normalize the data
				0,          // 0 = move forward size * sizeof(type) each iteration to get the next position
				0,          // start at the beginning of the buffer
			);

			gl.uniform2f(resolutionLocation, gl.canvas.width, gl.canvas.height);
			gl.uniform2f(mouseLocation, m.x, -m.y);
			gl.uniform1f(timeLocation, time);

			gl.drawArrays(
				gl.TRIANGLES,
				0,     // offset
				6,     // num vertices to process
			);
		}());

		return () => {
			cancelAnimationFrame(frame);
		};

	});
</script>

<style>
	canvas {
		width: 100%;
		height: 100%;
		background-color: #333;
	}
</style>

<canvas on:mousemove={handleMousemove}
	bind:this={canvas}
	bind:clientWidth={clientWidth}
	bind:clientHeight={clientHeight} 
	width={canvasWidth}
	height={canvasHeight}
></canvas>
