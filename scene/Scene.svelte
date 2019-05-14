<script>
	import { setContext, onMount, onDestroy } from 'svelte';
	import { writable } from 'svelte/store';
	import { RENDERER, LAYER, PARENT, CAMERA, create_layer } from '../internal/index.mjs';
	import * as mat4 from 'gl-matrix/mat4';
	import * as vec3 from 'gl-matrix/vec3';

	export let background = [1, 1, 1, 1];

	let canvas;
	let w;
	let h;

	let gl;
	let draw;
	let camera_stores = {
		matrix: writable(),
		view: writable(),
		projection: writable()
	};

	const width = writable(1);
	const height = writable(1);

	const root_layer = create_layer(0, invalidate);

	const default_camera = { /* TODO */ };
	let camera = default_camera;
	const num_lights = 8;

	const meshes = [];

	// lights
	const lights = {
		ambient: [],
		directional: [],
		point: []
	};

	let update_scheduled = false;
	let resolved = Promise.resolve();

	function invalidate() {
		if (!update_scheduled) {
			update_scheduled = true;
			resolved.then(draw);
		}
	}

	function add_to(array) {
		return fn => {
			array.push(fn);
			invalidate();

			onDestroy(() => {
				const i = array.indexOf(fn);
				if (~i) array.splice(i, 1);
				invalidate();
			});
		}
	}

	const scene = {
		add_camera: _camera => {
			if (camera && camera !== default_camera) {
				throw new Error(`A scene can only have one camera`);
			}

			camera = _camera;
			invalidate();

			// TODO this is garbage
			camera_stores.projection.set(camera.projection);
			camera_stores.view.set(camera.view);

			onDestroy(() => {
				camera = default_camera;
				invalidate();
			});
		},

		add_directional_light: add_to(lights.directional),
		add_point_light: add_to(lights.point),
		add_ambient_light: add_to(lights.ambient),

		invalidate,

		camera_matrix: camera_stores.matrix,
		view: camera_stores.view,
		projection: camera_stores.projection,
		width,
		height
	};

	setContext(RENDERER, scene);
	setContext(LAYER, root_layer);

	const origin = mat4.identity(mat4.create());
	const ctm = writable(origin);
	setContext(PARENT, {
		get_matrix_world: () => origin,
		ctm: { subscribe: ctm.subscribe }
	});

	// TEMP
	export let blend = undefined;
	$: (blend, draw && invalidate());

	onMount(() => {
		scene.canvas = canvas;
		gl = scene.gl = canvas.getContext('webgl');

		const extensions = [
			'OES_element_index_uint',
			'OES_standard_derivatives'
		];

		extensions.forEach(name => {
			const ext = gl.getExtension(name);
			console.log(ext);
			// if (!gl.getExtension(ext)) {
			// 	throw new Error(`Unsupported extension: ${ext}`);
			// }
		});

		draw = () => {
			if (!camera) return; // TODO make this `!ready` or something instead

			update_scheduled = false;

			gl.clearColor(...background);
			gl.clear(gl.COLOR_BUFFER_BIT | gl.DEPTH_BUFFER_BIT);

			gl.enable(gl.CULL_FACE);
			gl.enable(gl.DEPTH_TEST);

			gl.enable(gl.BLEND);
			gl.blendFunc(gl.SRC_ALPHA, gl.ONE_MINUS_SRC_ALPHA);
			// gl.blendFunc(gl[sfactor], gl[dfactor]);
			// gl.blendFuncSeparate(gl.ZERO, gl.SRC_COLOR, gl.ZERO, gl.SRC_ALPHA);

			// for overlays
			camera_stores.matrix.set(camera.matrix);
			camera_stores.view.set(camera.view);
			camera_stores.projection.set(camera.projection);

			// calculate total ambient light
			const ambient_light = lights.ambient.reduce((total, { color, intensity }) => {
				return [
					Math.min(total[0] + color[0] * intensity, 1),
					Math.min(total[1] + color[1] * intensity, 1),
					Math.min(total[2] + color[2] * intensity, 1)
				];
			}, new Float32Array([0, 0, 0]));

			let previous_program_info;

			function render_mesh({ model, model_inverse_transpose, geometry, material, program_info }) {
				// TODO...
				// if (material.blend === 'multiply') {
				// 	gl.blendFuncSeparate(gl[blend.srgb], gl[blend.drgb], gl[blend.salpha], gl[blend.dalpha]);
				// } else {
					gl.blendFunc(gl.ONE, gl.ONE_MINUS_SRC_ALPHA);
				// }

				if (program_info !== previous_program_info) {
					gl.useProgram(program_info.program);

					// set built-ins
					gl.uniform3fv(program_info.uniform_locations.AMBIENT_LIGHT, ambient_light);

					if (program_info.uniform_locations.DIRECTIONAL_LIGHTS) {
						for (let i = 0; i < num_lights; i += 1) {
							const light = lights.directional[i];
							if (!light) break;

							gl.uniform3fv(program_info.uniform_locations.DIRECTIONAL_LIGHTS[i].direction, light.direction);
							gl.uniform3fv(program_info.uniform_locations.DIRECTIONAL_LIGHTS[i].color, light.color);
							gl.uniform1f(program_info.uniform_locations.DIRECTIONAL_LIGHTS[i].intensity, light.intensity);
						}
					}

					if (program_info.uniform_locations.POINT_LIGHTS) {
						for (let i = 0; i < num_lights; i += 1) {
							const light = lights.point[i];
							if (!light) break;

							gl.uniform3fv(program_info.uniform_locations.POINT_LIGHTS[i].location, light.location);
							gl.uniform3fv(program_info.uniform_locations.POINT_LIGHTS[i].color, light.color);
							gl.uniform1f(program_info.uniform_locations.POINT_LIGHTS[i].intensity, light.intensity);
						}
					}

					gl.uniform3fv(program_info.uniform_locations.CAMERA_WORLD_POSITION, camera.world_position);
					gl.uniformMatrix4fv(program_info.uniform_locations.VIEW, false, camera.view);
					gl.uniformMatrix4fv(program_info.uniform_locations.PROJECTION, false, camera.projection);

					previous_program_info = program_info;
				}

				// set mesh-specific built-in uniforms
				gl.uniformMatrix4fv(program_info.uniform_locations.MODEL, false, model);
				gl.uniformMatrix4fv(program_info.uniform_locations.MODEL_INVERSE_TRANSPOSE, false, model_inverse_transpose);

				// set material-specific built-in uniforms
				material._set_uniforms(gl, program_info.uniforms, program_info.uniform_locations);

				// set attributes
				geometry._set_attributes(gl);

				// draw
				if (geometry.index) {
					gl.bindBuffer(gl.ELEMENT_ARRAY_BUFFER, geometry.buffers.__index);
					gl.drawElements(gl[geometry.primitive], geometry.index.length, gl.UNSIGNED_INT, 0);
				} else {
					const primitiveType = gl.TRIANGLES;
					const offset = 0;
					const { position } = geometry.attributes;
					const count = position.count;
					gl.drawArrays(primitiveType, offset, count);
				}
			}

			function render_layer(layer) {
				if (layer.needs_sort) {
					layer.child_layers.sort((a, b) => a.index - b.index);
					layer.needs_sort = false;
				}

				gl.depthMask(true);
				gl.clearDepth(1.0);
				gl.clear(gl.DEPTH_BUFFER_BIT);

				const transparent = [];

				for (let i = 0; i < layer.meshes.length; i += 1) {
					const mesh = layer.meshes[i];

					if (mesh.material.alpha < 1) {
						transparent.push(mesh);
					} else {
						render_mesh(mesh);
					}
				}

				// TODO sort transparent meshes, furthest to closest
				gl.depthMask(false);
				transparent.forEach(render_mesh);

				for (let i = 0; i < layer.child_layers.length; i += 1) {
					render_layer(layer.child_layers[i]);
				}
			}

			render_layer(root_layer);
		};
	});

	// TODO just set a dirty flag if dimensions change?
	$: if (gl) {
		const DPR = window.devicePixelRatio || 1;
		canvas.width = $width * DPR;
		canvas.height = $height * DPR;
		gl.viewport(0, 0, $width * DPR, $height * DPR);
		draw();
	}
</script>

<style>
	.container, canvas {
		position: relative;
		width: 100%;
		height: 100%;
		display: block;
	}
</style>

<div class="container" bind:clientWidth={$width} bind:clientHeight={$height}>
	<canvas bind:this={canvas}></canvas>

	{#if gl}
		<slot></slot>
	{/if}
</div>