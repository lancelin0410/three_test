<template>
  <div id="app" ref="container">
		<canvas id="renderCanvas" ref="renderCanvas" />
		<!-- <div>{{ camera ? camera }}</div> -->
  </div>
</template>

<script>
import * as THREE from 'three';
import { OrbitControls } from "three/examples/jsm/controls/OrbitControls";
import { MeshLine, MeshLineMaterial, MeshLineRaycast } from 'three.meshline';
import * as dat from 'dat.gui';

export default {
  name: 'App',
	data() {
		return {
			isUserInteracting: false,
			// onPointerDownMouseX: 0, 
			// onPointerDownMouseY: 0,
			// lon: 0, 
			// onPointerDownLon: 0,
			// lat: 0, 
			// onPointerDownLat: 0,
			// phi: 0, 
			// theta: 0,
			guiController: {
				cameraFOV: null
			},
			guiInfo: {
				cameraFOV: 110
			},
			scene: null,
			camera: null,
			geometry: null,
			texture: null,
			material: null,
			mesh: null,
			renderer: null,
			controls: null
		}
	},
	created() {},
	mounted() {
		this.initThree();
		this.animate();

		this.initGUI();
	},
	methods: {
		initGUI() {
			const gui = new dat.GUI({ name: "threeJS GUI" });
			this.guiController.cameraFOV = gui.add(this.guiInfo, "cameraFOV", 10, 110, 1 );

			this.guiController.cameraFOV.onChange(value => {
				this.camera.fov = value;
				this.camera.updateProjectionMatrix();
			});
		},
		initThree() {
			// init scene & camera
			this.scene = new THREE.Scene();
			this.camera = new THREE.PerspectiveCamera( 110, window.innerWidth / window.innerHeight, 1, 1100 );
			// this.camera = new THREE.OrthographicCamera( -window.innerWidth/2,  window.innerWidth/2, window.innerHeight/2, -window.innerHeight/2, 1, 1100 );
			this.camera.position = new THREE.Vector3(0, 0, 1);

			// 半球光
			// let light = new THREE.HemisphereLight(0xffffff);
			// light.position.set(0, 40, 0);
			// this.scene.add(light);
			// let helper = new THREE.HemisphereLightHelper( light, 5 );
			// this.scene.add(helper);

			//平行光
			// light = new THREE.DirectionalLight(0xffffff);
			// light.position.set(0, 40, -10);
			// this.scene.add(light);
			// helper = new THREE.DirectionalLightHelper( light, 5 );
			// this.scene.add(helper);

			// helper	
			const cameraHelper = new THREE.CameraHelper( this.camera );
			this.scene.add(cameraHelper);
			const axisHelper = new THREE.AxesHelper(1100);
			this.scene.add(axisHelper);

			// 建立 球體 & 全景貼圖
			this.geometry = new THREE.SphereGeometry( 500, 60, 40 );
			// invert the geometry on the x-axis so that all of the faces point inward
			this.geometry.scale( -1, 1, 1 );
			// this.texture = new THREE.TextureLoader().load('https://storage.googleapis.com/map_panorama/demo/pano_road/7/202008/005200-X-2020071707502152-0001849.jpg');
			this.texture = new THREE.TextureLoader().load('https://storage.googleapis.com/map_panorama/demo/pano_walk/9/202209/Z111_WALK_20220920_144444_50.jpg');
			this.material = new THREE.MeshBasicMaterial( { map: this.texture } );
			// this.material = new THREE.MeshBasicMaterial( { color: 0x00ff00 } );
			this.mesh = new THREE.Mesh( this.geometry, this.material );
			this.mesh.rotateY(-Math.PI/2);
			this.scene.add( this.mesh );

			// init renderer
			this.renderer = new THREE.WebGLRenderer({ canvas: this.$refs.renderCanvas, preserveDrawingBuffer: true });
			this.renderer.setPixelRatio( window.devicePixelRatio );
			this.renderer.setSize(window.innerWidth, window.innerHeight);

			this.controls = new OrbitControls(this.camera, this.renderer.domElement);
			// this.controls.enableZoom = true; //啟用縮放
			// this.controls.enableDamping = true; // 啟用阻尼效果
			// this.controls.dampingFactor = 0.25; // 阻尼系數
			// this.controls.autoRotate = true; //啟用自動旋轉
			// this.controls.target.set(-500, 0, 0);
			// this.controls.minDistance = 0; 
			// this.controls.maxDistance = 1;
			this.controls.update();

			// 監聽
			this.$refs.container.style.touchAction = 'none';
			// this.$refs.container.addEventListener( 'pointerdown', this.onPointerDown );
			document.addEventListener( 'wheel', this.onDocumentMouseWheel );
			// document.addEventListener( 'dragover', event => {
			// 	event.preventDefault();
			// 	event.dataTransfer.dropEffect = 'copy';
			// });

			// document.addEventListener( 'dragenter', () => { document.body.style.opacity = 0.5; });
			// document.addEventListener( 'dragleave', () => { document.body.style.opacity = 1; });
			// document.addEventListener( 'drop', event => {
			// 	event.preventDefault();

			// 	const reader = new FileReader();
			// 	reader.addEventListener( 'load', event => {
			// 		material.map.image.src = event.target.result;
			// 		material.map.needsUpdate = true;
			// 	} );

			// 	reader.readAsDataURL( event.dataTransfer.files[ 0 ] );
			// 	document.body.style.opacity = 1;
			// } );

			window.addEventListener( 'resize', this.onWindowResize );
			this.draw2Line();
		},
		onWindowResize() {
			this.camera.aspect = window.innerWidth / window.innerHeight;
			this.camera.updateProjectionMatrix();
			this.renderer.setSize( window.innerWidth, window.innerHeight );
		},
		draw2Line() {
			let points = [];
			const limit = 3 * Math.PI / 180;
			// const limit = 0;
			for (let j = 0; j <= Math.PI; j += Math.PI / 100) {
			// for (let j = 0 + limit; j <= Math.PI - limit; j += Math.PI / 100) {
				if(j <= limit) points.push(new THREE.Vector3(0, -600 * Math.sin(j), 600));
				else if (j >= Math.PI - limit) points.push(new THREE.Vector3(0, -600 * Math.sin(j), -600));
				else points.push(new THREE.Vector3(0, -497 * Math.sin(j), 497 * Math.cos(j)));
			}

			this.drawLine(points, '#03A9F4');

			// let points2 = [
			// 	new THREE.Vector3(0, -100, 500),
			// 	new THREE.Vector3(0, -100, -500)
			// ];
			// this.drawLine(points2, '#FF9800');
		},
		drawLine(points, color) {
			const geometry = new THREE.BufferGeometry().setFromPoints(points);
			// geometry.rotateZ( 35 * Math.PI / 180);
			const line = new MeshLine();
			line.setGeometry(geometry);

			const material = new MeshLineMaterial({ map: this.texture, color: color, lineWidth: 25, transparent: true, opacity: 0.5 } );

			const mesh = new THREE.Mesh(line, material);
			this.scene.add(mesh);

		},
		// onPointerDown(event) {
		// 	if ( event.isPrimary === false ) return;
		// 	this.isUserInteracting = true;
		// 	this.onPointerDownMouseX = event.clientX;
		// 	this.onPointerDownMouseY = event.clientY;
		// 	this.onPointerDownLon = this.lon;
		// 	this.onPointerDownLat = this.lat;

		// 	document.addEventListener( 'pointermove', this.onPointerMove );
		// 	document.addEventListener( 'pointerup', this.onPointerUp );
		// },
		// onPointerMove(event) {
		// 	if ( event.isPrimary === false ) return;

		// 	this.lon = ( this.onPointerDownMouseX - event.clientX ) * 0.1 + this.onPointerDownLon;
		// 	this.lat = ( event.clientY - this.onPointerDownMouseY ) * 0.1 + this.onPointerDownLat;
		// },
		// onPointerUp(event) {
		// 	if ( event.isPrimary === false ) return;
		// 	this.isUserInteracting = false;

		// 	document.removeEventListener( 'pointermove', this.onPointerMove );
		// 	document.removeEventListener( 'pointerup', this.onPointerUp );

		// },
		onDocumentMouseWheel(event) {
			const fov = this.camera.fov + event.deltaY * 0.05;
			this.guiInfo.cameraFOV = this.camera.fov = THREE.MathUtils.clamp( fov, 10, 110 );
			this.camera.updateProjectionMatrix();
			this.guiController.cameraFOV.updateDisplay() 
		},
		animate() {
			requestAnimationFrame( this.animate );

			// this.update();

			this.controls.update();
			this.renderer.render( this.scene, this.camera );
		},
		update() {
			this.lat = Math.max( - 85, Math.min( 85, this.lat ) );
			this.phi = THREE.MathUtils.degToRad( 90 - this.lat );
			this.theta = THREE.MathUtils.degToRad( this.lon );

			const x = 500 * Math.sin( this.phi ) * Math.cos( this.theta );
			const y = 500 * Math.cos( this.phi );
			const z = 500 * Math.sin( this.phi ) * Math.sin( this.theta );

			this.camera.lookAt( x, y, z );
			this.renderer.render( this.scene, this.camera );
		}
	}
}
</script>

<style lang="sass">
#app 
	font-family: Avenir, Helvetica, Arial, sans-serif
	-webkit-font-smoothing: antialiased
	-moz-osx-font-smoothing: grayscale
	text-align: center
	color: #2c3e50
	margin-top: 60px
</style>
