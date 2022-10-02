<template>
  <div id="app" ref="container">
		<canvas id="renderCanvas" ref="renderCanvas" />
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
			sceneOrtho: null,
			cameraOrtho: null,
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

			//TODO: test
			this.cameraOrtho = new THREE.OrthographicCamera(-window.innerWidth / 2, window.innerWidth / 2, window.innerHeight / 2, -window.innerHeight / 2, 1, 10);  
			this.cameraOrtho.position = new THREE.Vector3(0, 0, 10);
			this.sceneOrtho = new THREE.Scene();

			let _labels = [];
			const _labelsOri =  [
				{ position: { lon: -72.00, lat: 9.00 }, logoUrl: '', text: '蓝窗户' },
				{ position: { lon: 114.12, lat: 69.48 }, logoUrl: '', text: '一片云彩' },
				{ position: { lon: 132.48, lat: -12.24 }, logoUrl: '', text: '大海' }
			];


			for (var i = 0; i < _labelsOri.length; i++) {
				_labels.push(this.createLabelSprite(this.sceneOrtho, _labelsOri[i].text, _labelsOri[i].position));
				var wp = this.geoPosition2World(_labels[i].pos.lon, _labels[i].pos.lat);
				var sp = this.worldPosition2Screen(wp, this.camera);
				var test = wp.clone();
				test.project(this.camera);
				if (test.x > -1 && test.x < 1 && test.y > -1 && test.y < 1 && test.z > -1 && test.z < 1) {
					var metrics = _labels[i].context.measureText(_labels[i].name);
					var width = metrics.width * 3.5;
					_labels[i].sprite.scale.set(400, 150, 1.0);
					_labels[i].sprite.position.set(sp.x + width, sp.y - 40, 1);
				}
				else {
					_labels[i].sprite.scale.set(1.0, 1.0, 1.0);
					_labels[i].sprite.position.set(0, 0, 0);
				}
			}
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
			const cameraOrthoHelper = new THREE.CameraHelper( this.cameraOrtho );
			this.scene.add(cameraOrthoHelper);
			const axisHelper = new THREE.AxesHelper(1100);
			this.scene.add(axisHelper);

			// 建立 球體 & 全景貼圖
			this.geometry = new THREE.SphereGeometry( 500, 60, 40 );
			// invert the geometry on the x-axis so that all of the faces point inward
			this.geometry.scale( -1, 1, 1 );
			this.texture = new THREE.TextureLoader().load('https://storage.googleapis.com/map_panorama/demo/pano_road/7/202008/005200-X-2020071707502152-0001849.jpg');
			// this.texture = new THREE.TextureLoader().load('https://storage.googleapis.com/map_panorama/demo/pano_walk/9/202209/Z111_WALK_20220920_144444_50.jpg');
			this.material = new THREE.MeshBasicMaterial( { map: this.texture } );
			// this.material = new THREE.MeshBasicMaterial( { color: 0x00ff00 } );
			this.mesh = new THREE.Mesh( this.geometry, this.material );
			this.mesh.rotateY(-Math.PI/2);
			this.scene.add( this.mesh );

			// init renderer
			this.renderer = new THREE.WebGLRenderer({ canvas: this.$refs.renderCanvas });
			this.renderer.setPixelRatio( window.devicePixelRatio );
			this.renderer.setSize(window.innerWidth, window.innerHeight);
			this.renderer.autoClear = false;

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
		// 创建文字标记
		createLabelSprite(sceneOrtho, name, position) {
			const canvas1 = document.createElement('canvas');
			const context1 = canvas1.getContext('2d');
			const metrics = context1.measureText(name);
			const width = metrics.width * 1.5;
			context1.font = "20px 微軟正黑體";
			context1.fillStyle = "rgba(0,0,0,0.95)";
			context1.fillRect(0, 0, width + 8, 20 + 8);
			context1.fillStyle = "rgba(0,0,0,0.2)";
			context1.fillRect(2, 2, width + 4, 20 + 4);
			context1.fillStyle = "rgba(255,255,255,0.95)";
			context1.fillText(name, 4, 20);
			const texture1 = new THREE.Texture(canvas1);
			texture1.needsUpdate = true;
			const spriteMaterial = new THREE.SpriteMaterial({ map: texture1 });
			const sprite1 = new THREE.Sprite(spriteMaterial);
			sprite1.scale.set(1.0, 1.0, 1.0);
			sprite1.position.set(0, 0, 0);
			sprite1.name = name;
			const label = {
				name: name,
				pos: position,
				canvas: canvas1,
				context: context1,
				texture: texture1,
				sprite: sprite1
			};
			sceneOrtho.add(label.sprite);
			return label;
		},
		calPosition() {
			_lat = Math.max(-85, Math.min(85, _lat));
			const phi = THREE.MathUtils.degToRad(90 - _lat);
			const theta = THREE.MathUtils.degToRad(_lon);
			this.camera.target.x = _pRadius * Math.sin(phi) * Math.cos(theta);
			this.camera.target.y = _pRadius * Math.cos(phi);
			this.camera.target.z = _pRadius * Math.sin(phi) * Math.sin(theta);
			this.camera.lookAt(this.camera.target);
		},
		addSprites() {
			if (typeof (_sprites) != "undefined") {
				for (var i = 0; i < _sprites.length; i++) {
					var wp = this.geoPosition2World(_sprites[i].pos.lon, _sprites[i].pos.lat);
					var sp = this.worldPosition2Screen(wp, this.camera);
					var test = wp.clone();
					test.project(this.camera);
					if (test.x > -1 && test.x < 1 && test.y > -1 && test.y < 1 && test.z > -1 && test.z < 1) {
						_sprites[i].sprite.scale.set(32, 32, 32);
						_sprites[i].sprite.position.set(sp.x, sp.y, 1);
					}
					else {
						_sprites[i].sprite.scale.set(1.0, 1.0, 1.0);
						_sprites[i].sprite.position.set(0, 0, 0);
					}
				}
			}
			if (typeof (_labels) != "undefined") {
				for (var i = 0; i < _labels.length; i++) {
					var wp = this.geoPosition2World(_labels[i].pos.lon, _labels[i].pos.lat);
					var sp = this.worldPosition2Screen(wp, this.camera);
					var test = wp.clone();
					test.project(this.camera);
					if (test.x > -1 && test.x < 1 && test.y > -1 && test.y < 1 && test.z > -1 && test.z < 1) {
						var metrics = _labels[i].context.measureText(_labels[i].name);
						var width = metrics.width * 3.5;
						_labels[i].sprite.scale.set(400, 150, 1.0);
						_labels[i].sprite.position.set(sp.x + width, sp.y - 40, 1);
					}
					else {
						_labels[i].sprite.scale.set(1.0, 1.0, 1.0);
						_labels[i].sprite.position.set(0, 0, 0);
					}
				}
			}
		},
		// 创建图片标记
		createSprite(position, url, name) {  
			const textureLoader = new THREE.TextureLoader();  
			const ballMaterial = new THREE.SpriteMaterial({ map: textureLoader.load(url)  });  
			const sp = {    
				pos: position,    
				name: name,    
				sprite: new THREE.Sprite(ballMaterial)  
			};  
			sp.sprite.scale.set(32, 32, 1.0);  
			sp.sprite.name = name;  
			this.sceneOrtho.add(sp.sprite);  
			return sp;
		},
		geoPosition2World(lon, lat) {
			const _pRadius = 1000;
			lat = Math.max(-85, Math.min(85, lat));
			const phi = THREE.MathUtils.degToRad(90 - lat);
			const theta = THREE.MathUtils.degToRad(lon);

			const result = {
				x: _pRadius * Math.sin(phi) * Math.cos(theta),
				y: _pRadius * Math.cos(phi),
				z: _pRadius * Math.sin(phi) * Math.sin(theta)
			}
			return new THREE.Vector3(result.x, result.y, result.z);
		},
		worldPosition2Screen(world_vector, camera) {
			const vector = world_vector.clone();
			vector.project(camera);
			const result = {
				x: Math.round((vector.x + 1) * window.innerWidth / 2 - window.innerWidth / 2),
				y: Math.round(window.innerHeight / 2 - (-vector.y + 1) * window.innerHeight / 2),
				z: 0
			};
			return new THREE.Vector3(result.x, result.y, result.z);
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
			this.renderer.render( this.sceneOrtho, this.cameraOrtho );
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
			this.renderer.render( this.sceneOrtho, this.cameraOrtho );
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
