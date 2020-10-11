<script>

	/*

	WebGL utility methods from :
	https://webglfundamentals.org/webgl/resources/webgl-utils.js

	* Copyright 2012, Gregg Tavares.
	* All rights reserved.
	*
	* Redistribution and use in source and binary forms, with or without
	* modification, are permitted provided that the following conditions are
	* met:
	*
	*     * Redistributions of source code must retain the above copyright
	* notice, this list of conditions and the following disclaimer.
	*     * Redistributions in binary form must reproduce the above
	* copyright notice, this list of conditions and the following disclaimer
	* in the documentation and/or other materials provided with the
	* distribution.
	*     * Neither the name of Gregg Tavares. nor the names of his
	* contributors may be used to endorse or promote products derived from
	* this software without specific prior written permission.
	*
	* THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS
	* "AS IS" AND ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT
	* LIMITED TO, THE IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR
	* A PARTICULAR PURPOSE ARE DISCLAIMED. IN NO EVENT SHALL THE COPYRIGHT
	* OWNER OR CONTRIBUTORS BE LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL,
	* SPECIAL, EXEMPLARY, OR CONSEQUENTIAL DAMAGES (INCLUDING, BUT NOT
	* LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS OR SERVICES; LOSS OF USE,
	* DATA, OR PROFITS; OR BUSINESS INTERRUPTION) HOWEVER CAUSED AND ON ANY
	* THEORY OF LIABILITY, WHETHER IN CONTRACT, STRICT LIABILITY, OR TORT
	* (INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE USE
	* OF THIS SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.
	*/

	import { onMount } from 'svelte';

	let canvas;

	export let width;
	export let height;

	let m = { x: 0, y: 0 };
	let time = 0;

	function handleMousemove(event) {
		m.x = event.clientX / 6;
		m.y = event.clientY / 6;
	}

	onMount(() => {
		const gl = canvas.getContext('webgl');
		let frame;

		if (!gl) {
			return; // Fallback options for no webgl support
		}

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



		const defaultShaderType = [
			'VERTEX_SHADER',
			'FRAGMENT_SHADER',
		];

		/**
		 * Creates a program from 2 sources.
		 *
		 * @param {WebGLRenderingContext} gl The WebGLRenderingContext
		 *        to use.
		 * @param {string[]} shaderSourcess Array of sources for the
		 *        shaders. The first is assumed to be the vertex shader,
		 *        the second the fragment shader.
		 * @param {string[]} [opt_attribs] An array of attribs names. Locations will be assigned by index if not passed in
		 * @param {number[]} [opt_locations] The locations for the. A parallel array to opt_attribs letting you assign locations.
		 * @param {module:webgl-utils.ErrorCallback} opt_errorCallback callback for errors. By default it just prints an error to the console
		 *        on error. If you want something else pass an callback. It's passed an error message.
		 * @return {WebGLProgram} The created program.
		 * @memberOf module:webgl-utils
		 */
		function createProgramFromSources(
			gl, shaderSources, opt_attribs, opt_locations, opt_errorCallback) {
			const shaders = [];
			for (let ii = 0; ii < shaderSources.length; ++ii) {
			shaders.push(loadShader(
				gl, shaderSources[ii], gl[defaultShaderType[ii]], opt_errorCallback));
			}
			return createProgram(gl, shaders, opt_attribs, opt_locations, opt_errorCallback);
		}


		/**
		 * Loads a shader.
		 * @param {WebGLRenderingContext} gl The WebGLRenderingContext to use.
		 * @param {string} shaderSource The shader source.
		 * @param {number} shaderType The type of shader.
		 * @param {module:webgl-utils.ErrorCallback} opt_errorCallback callback for errors.
		 * @return {WebGLShader} The created shader.
		 */
		function loadShader(gl, shaderSource, shaderType, opt_errorCallback) {
			const errFn = opt_errorCallback || error;
			// Create the shader object
			const shader = gl.createShader(shaderType);

			// Load the shader source
			gl.shaderSource(shader, shaderSource);

			// Compile the shader
			gl.compileShader(shader);

			// Check the compile status
			const compiled = gl.getShaderParameter(shader, gl.COMPILE_STATUS);
			if (!compiled) {
			// Something went wrong during compilation; get the error
			const lastError = gl.getShaderInfoLog(shader);
			errFn('*** Error compiling shader \'' + shader + '\':' + lastError);
			gl.deleteShader(shader);
			return null;
			}

			return shader;
		}

		/**
		 * Creates a program, attaches shaders, binds attrib locations, links the
		 * program and calls useProgram.
		 * @param {WebGLShader[]} shaders The shaders to attach
		 * @param {string[]} [opt_attribs] An array of attribs names. Locations will be assigned by index if not passed in
		 * @param {number[]} [opt_locations] The locations for the. A parallel array to opt_attribs letting you assign locations.
		 * @param {module:webgl-utils.ErrorCallback} opt_errorCallback callback for errors. By default it just prints an error to the console
		 *        on error. If you want something else pass an callback. It's passed an error message.
		 * @memberOf module:webgl-utils
		 */
		function createProgram(
			gl, shaders, opt_attribs, opt_locations, opt_errorCallback) {
			const errFn = opt_errorCallback || error;
			const program = gl.createProgram();
			shaders.forEach(function(shader) {
			gl.attachShader(program, shader);
			});
			if (opt_attribs) {
			opt_attribs.forEach(function(attrib, ndx) {
				gl.bindAttribLocation(
					program,
					opt_locations ? opt_locations[ndx] : ndx,
					attrib);
			});
			}
			gl.linkProgram(program);

			// Check the link status
			const linked = gl.getProgramParameter(program, gl.LINK_STATUS);
			if (!linked) {
				// something went wrong with the link
				const lastError = gl.getProgramInfoLog(program);
				errFn('Error in program linking:' + lastError);

				gl.deleteProgram(program);
				return null;
			}
			return program;
		}

		/**
		 * Wrapped logging function.
		 * @param {string} msg The message to log.
		 */
		function error(msg) {
			if (topWindow.console) {
			if (topWindow.console.error) {
				topWindow.console.error(msg);
			} else if (topWindow.console.log) {
				topWindow.console.log(msg);
			}
			}
		}

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
	width={width}
	height={height}
></canvas>
