<script>
	import Shader from "./Shader.svelte";

	let scale = 1;

	let fragmentShader = `
		precision highp float;

		uniform vec2 u_resolution;
		uniform vec2 u_mouse;
		uniform float u_time;

		void main() {
			gl_FragColor = vec4(fract((gl_FragCoord.xy - u_mouse) / u_resolution), fract(u_time), 1);
		}
	`;

</script>
 
<Shader {scale} {fragmentShader} />