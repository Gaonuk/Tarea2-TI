<script>
	import L, { marker } from "leaflet";
	import * as markerIcons from "./markers.js";
	import PlanePopup from "./PlanePopup.svelte";
	let map;

	const initialView = [-37.6, -64.5795];
	function createMap(container) {
		let m = L.map(container, { preferCanvas: true }).setView(
			initialView,
			6
		);
		L.tileLayer(
			"https://{s}.basemaps.cartocdn.com/rastertiles/voyager/{z}/{x}/{y}{r}.png",
			{
				attribution: `&copy;<a href="https://www.openstreetmap.org/copyright" target="_blank">OpenStreetMap</a>,
	        &copy;<a href="https://carto.com/attributions" target="_blank">CARTO</a>`,
				subdomains: "abcd",
				maxZoom: 14,
			}
		).addTo(m);

		return m;
	}

	import io from "socket.io-client";
	import { fade } from "svelte/transition";

	const socket = io("ws://tarea-3-websocket.2021-1.tallerdeintegracion.cl", {
		path: "/flights",
	});

	const placeholder = "Escribe tu mensaje aqui...";
	const greeting = `Acabas de unirte al chat! Usa '/nick tu_nickname' para establecer tu nickname!`;
	let messages = [];
	let message = "";
	let name = "Torre de Control";

	const planes = {};
	let flights = [];

	socket.on("FLIGHTS", (flight) => {
		flights = flights.concat(flight);
		console.log(flights);
		addTravels(flights);
	});

	let coords = [];

	function addTravels(flights) {
		let newLayer;
		for (let flight of flights) {
			coords.push(flight.origin);
			let m = createMarker(flight.origin);
			markerLayers.addLayer(m);
			coords.push(flight.destination);
			let m2 = createMarker(flight.destination);
			markerLayers.addLayer(m2);
			newLayer = createLines(flight.origin, flight.destination);
			newLayer.addTo(map);
		}
	}

	socket.emit("FLIGHTS");

	socket.on("POSITION", (position) => {
		planes[position.code] = position;
		updatePositions(planes);
		// console.log(planes)
	});

	let realLayer = L.layerGroup();
	const planeMarkers = {};

	function updatePositions(planes) {
		let codes = Object.keys(planes);
		console.log(codes);
		for (let code of codes) {
			let planeIcon;
			if (!planeMarkers[code]) {
				console.log(planes[code]);
				planeIcon = createPlane(planes[code].position, code);
				planeMarkers[code] = planeIcon;
				console.log("first for:" + code);
			} else {
				planeIcon = planeMarkers[code];
				planeIcon.setLatLng(planes[code].position);
				planeMarkers[code] = planeIcon;
				console.log("reuse");
			}
			let dot = createDot(planes[code].position);
			realLayer.addLayer(dot);
			realLayer.addLayer(planeIcon);
		}
		realLayer.addTo(map);
	}

	socket.on("CHAT", function (data) {
		console.log(data);
		messages = messages.concat(data);
		updateScroll();
	});

	function markerIcon() {
		let html = `<div class="map-marker"><div>${markerIcons.location}</div></div>`;
		return L.divIcon({
			html,
			className: "map-marker",
		});
	}

	function markerPlane() {
		let html = `<div class="map-marker"><div>${markerIcons.plane}</div></div>`;
		return L.divIcon({
			html,
			className: "map-marker",
		});
	}

	function createMarker(loc) {
		let icon = markerIcon();
		let marker = L.marker(loc, { icon });
		return marker;
	}

	function createDot(loc) {
		return L.circle(loc, 200);
	}

	function createPlane(loc, code) {
		let icon = markerPlane();
		let marker = L.marker(loc, { icon });
		bindPopup(marker, (m) => {
			let c = new PlanePopup({
				target: m,
				props: {
					code,
				},
			});

			return c;
		});
		return marker;
	}

	function bindPopup(marker, createFn) {
		let popupComponent;
		marker.bindPopup(() => {
			let container = L.DomUtil.create("div");
			popupComponent = createFn(container);
			return container;
		});

		marker.on("popupclose", () => {
			if (popupComponent) {
				let old = popupComponent;
				popupComponent = null;
				// Wait to destroy until after the fadeout completes.
				setTimeout(() => {
					old.$destroy();
				}, 500);
			}
		});
	}

	function createLines(origin, destination) {
		var randomColor = Math.floor(Math.random() * 16777215).toString(16);
		console.log(randomColor);
		return L.polyline([origin, destination], {
			color: "#" + randomColor,
			opacity: 0.5,
			dashArray: "10",
		});
	}

	let markerLayers;
	let lineLayers;
	function mapAction(container) {
		map = createMap(container);

		markerLayers = L.layerGroup();
		for (let location of coords) {
			let m = createMarker(location);
			markerLayers.addLayer(m);
		}

		markerLayers.addTo(map);
	}

	function handleSubmit() {
		message = message.trim();

		if (message == "") {
			return;
		}

		let messageString;

		if (message.slice(0, 5) == "/nick") {
			let newName = message.slice(6);
			messageString = `Server: ${name} changed their nickname to ${newName}`;
			name = newName;
		}

		let payload = {
			name,
			message,
		};

		// messages = messages.concat(messageString);
		socket.emit("CHAT", payload);

		updateScroll();

		message = "";
	}

	function updateScroll() {
		const chatWindow = document.getElementById("chatWindow");
		setTimeout(() => {
			chatWindow.scrollTop = chatWindow.scrollHeight;
		}, 0);
	}
</script>

<!-- <svelte:window on:resize={resizeMap} /> -->

<link
	rel="stylesheet"
	href="https://unpkg.com/leaflet@1.6.0/dist/leaflet.css"
	integrity="sha512-xwE/Az9zrjBIphAcBb3F6JVqxf46+CDLwfLMHloNu6KEQCAWi6HcDUbeOfBIptF7tcCzusKFjFw2yuvEpDL9wQ=="
	crossorigin=""
/>

<!-- <div class="map" style="height:80%;width:80%" use:mapAction /> -->
<div class="grid-container" style="height:100%;width:100%">
	<div class="map" use:mapAction />
	<div class="chat">
		<div id="chatWindow">
			<ul id="messages">
				<li>
					{greeting}
				</li>
				{#each messages as message}
					<li transition:fade>
						<p>
							<strong>{message.name}</strong>
							{new Date(message.date)
								.toISOString()
								.split("T")
								.join(" ")}
							<br />
							{message.message}
						</p>
					</li>
				{/each}
			</ul>
		</div>
		<form action="">
			<input
				id="m"
				autocomplete="off"
				{placeholder}
				bind:value={message}
			/>
			<button on:click|preventDefault={handleSubmit}>Enviar</button>
		</form>
	</div>
	<div class="flights">
		<div id="chatWindow">
			<ul id="messages">
				{#each flights as flight}
					<li transition:fade>
						<p>
							<strong>FLIGHT:</strong> {flight.code}
							<br />
							<strong>Airline:</strong> {flight.airline}
							<br />
							Origin: {flight.origin}
							<br />
							Destination: {flight.destination}
							<br />
							Plane: {flight.plane}
							<br />
							Seats: {flight.seats}
							<br />
							Passengers:
							{#each flight.passengers as passenger}
								<br />
								{passenger.name}
								{passenger.age}
							{/each}
						</p>
					</li>
				{/each}
			</ul>
		</div>
	</div>
</div>

<style>
	#chatWindow {
		height: 100%;
		width: 90%;
		border: 3px solid #01b3ed;
		margin: 3px;
		overflow: auto;
	}

	@media (min-height: 845px) {
		#chatWindow {
			height: 600px;
		}
	}

	#messages {
		align-self: center;
		list-style-type: none;
		margin: 0;
		padding: 0;
		font-size: 14px;
	}

	#messages li {
		padding: 5px 10px;
		/* border-radius: 1em; */
		width: auto;
		background: #fff8b8;
		color: #01b3ed;
		overflow-wrap: break-word;
	}

	/* #messages li:nth-child(odd) {
		background: #01b3ed;
		color: #fff8b8;
	} */

	.map :global(.marker-text) {
		width: 100%;
		text-align: center;
		font-weight: 600;
		background-color: #444;
		color: #eee;
		border-radius: 0.5rem;
	}

	.map :global(.map-marker) {
		width: 30px;
		transform: translateX(-50%) translateY(-25%);
	}

	.grid-container {
		display: grid;
		grid-template-columns: 1fr 1fr 1fr 1fr 1fr;
		grid-template-rows: 1fr 1fr 1fr;
		gap: 0px 0px;
		grid-template-areas:
			"flights map map map chat"
			"flights map map map chat"
			"flights map map map chat";
	}
	.map {
		grid-area: map;
	}
	.flights {
		grid-area: flights;
	}
	.chat {
		grid-area: chat;
	}
</style>
