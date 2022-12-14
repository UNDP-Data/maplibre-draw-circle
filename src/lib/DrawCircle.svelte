<script lang="ts">
	import Button, { Group, GroupItem, Label, Icon } from '@smui/button';
	import type { MenuComponentDev } from '@smui/menu';
	import Menu from '@smui/menu';
	import List, { Item, Text } from '@smui/list';
	import type { Map, MapMouseEvent } from 'maplibre-gl';
	import distance from '@turf/distance';
	import bearing from '@turf/bearing';
	import area from '@turf/area';
	import { destinationPoint as geodesyDestinationPoint } from 'geodesy-fn/src/spherical.js';
	import { circleFeatures } from './stores';

	export let map: Map;

	const SOURCE_LINE = 'draw-circle-controls-source-line';
	const LAYER_LINE = 'draw-circle-controls-layer-line';
	const LAYER_LINE_SYMBOL = 'draw-circle-controls-layer-line-symbol';
	const LAYER_SYMBOL = 'draw-circle-controls-layer-symbol';
	const SOURCE_SYMBOL = 'draw-circle-controls-source-symbol';
	const LAYER_SYMBOL_RADIUS = 'draw-circle-controls-layer-symbol-radius';
	const SOURCE_SYMBOL_RADIUS = 'draw-circle-controls-source-symbol-radius';
	const LAYER_CIRCLE = 'draw-circle-controls-layer-circle';
	const SOURCE_CIRCLE = 'draw-circle-controls-source-circle';
	const LAYER_CIRCLE_OUTLINE = 'draw-circle-controls-layer-circle-outline';

	let coordinatesCenter: number[][] = [];
	let coordinatesRadius: number[][] = [];
	let units = 'meters';
	let isDragging = false;
	let menu: MenuComponentDev;

	const drawStart = () => {
		if (map) {
			map.getCanvas().style.cursor = 'crosshair';
			clearFeatures();
			initFeatures();
			map.on('click', mapClickListener);
			map.on('mousemove', mapMoveListener);
			map.fire('drawcircle.on');
		}
	};

	const deleteFeatures = () => {
		if (!map) return;
		map.getCanvas().style.cursor = '';
		clearFeatures();
		initFeatures();
		map.off('click', mapClickListener);
		map.off('mousemove', mapMoveListener);
	};

	const mapClickListener = (event: MapMouseEvent) => {
		if (!map?.getSource(SOURCE_LINE) || !map?.getSource(SOURCE_SYMBOL)) {
			initFeatures();
		}
		const lnglat: number[] = [event.lngLat.lng, event.lngLat.lat];
		if (map) {
			if (coordinatesCenter.length === 0) {
				coordinatesCenter.push(lnglat);
			} else if (coordinatesRadius.length === 0) {
				coordinatesRadius.push(lnglat);
			} else {
				coordinatesRadius[coordinatesRadius.length - 1] = lnglat;
				map.off('click', mapClickListener);
				map.off('mousemove', mapMoveListener);
				map.getCanvas().style.cursor = '';
			}
		}
	};

	const mapMoveListener = (event: MapMouseEvent) => {
		if (!map) return;
		if (coordinatesCenter.length === 0) return;
		const lnglat: number[] = [event.lngLat.lng, event.lngLat.lat];
		if (coordinatesRadius.length === 0) {
			coordinatesRadius.push(lnglat);
		} else if (coordinatesRadius.length > 0) {
			coordinatesRadius[coordinatesRadius.length - 1] = lnglat;
		}
		// eslint-disable-next-line @typescript-eslint/ban-ts-comment
		// @ts-ignore
		map.getSource(SOURCE_LINE).setData(geoLineString([coordinatesCenter[0], coordinatesRadius[0]]));
		// eslint-disable-next-line @typescript-eslint/ban-ts-comment
		// @ts-ignore
		map.getSource(SOURCE_SYMBOL).setData(geoPoint(coordinatesCenter));
		// eslint-disable-next-line @typescript-eslint/ban-ts-comment
		// @ts-ignore
		map.getSource(SOURCE_SYMBOL_RADIUS).setData(geoPoint(coordinatesRadius));

		updateCircleFeature();
	};

	const clearFeatures = () => {
		if (map) {
			if (map.getLayer(LAYER_LINE)) map.removeLayer(LAYER_LINE);
			if (map.getLayer(LAYER_LINE_SYMBOL)) map.removeLayer(LAYER_LINE_SYMBOL);
			if (map.getLayer(LAYER_SYMBOL)) map.removeLayer(LAYER_SYMBOL);
			if (map.getLayer(LAYER_SYMBOL_RADIUS)) map.removeLayer(LAYER_SYMBOL_RADIUS);
			if (map.getLayer(LAYER_CIRCLE)) map.removeLayer(LAYER_CIRCLE);
			if (map.getLayer(LAYER_CIRCLE_OUTLINE)) map.removeLayer(LAYER_CIRCLE_OUTLINE);
			if (map.getSource(SOURCE_LINE)) map.removeSource(SOURCE_LINE);
			if (map.getSource(SOURCE_SYMBOL)) map.removeSource(SOURCE_SYMBOL);
			if (map.getSource(SOURCE_SYMBOL_RADIUS)) map.removeSource(SOURCE_SYMBOL_RADIUS);
			if (map.getSource(SOURCE_CIRCLE)) map.removeSource(SOURCE_CIRCLE);
			circleFeatures.update(() => []);
		}
	};

	const initFeatures = () => {
		coordinatesCenter = [];
		coordinatesRadius = [];
		if (map) {
			map.addSource(SOURCE_LINE, {
				type: 'geojson',
				data: geoLineString([])
			});
			map.addSource(SOURCE_SYMBOL, {
				type: 'geojson',
				data: geoPoint(coordinatesCenter)
			});
			map.addSource(SOURCE_SYMBOL_RADIUS, {
				type: 'geojson',
				data: geoPoint(coordinatesRadius)
			});

			map.addLayer({
				id: LAYER_LINE,
				type: 'line',
				source: SOURCE_LINE,
				paint: {
					'line-color': 'rgb(245,169,71)',
					'line-width': 2
				}
			});

			map.addLayer({
				id: LAYER_LINE_SYMBOL,
				type: 'symbol',
				source: SOURCE_LINE,
				layout: {
					'text-field': '{text}',
					// 'text-font': ['Roboto Medium'],
					'text-size': 14,
					'text-variable-anchor': ['top', 'bottom', 'left', 'right'],
					'text-radial-offset': 0.8,
					'text-justify': 'auto',
					'symbol-placement': 'line-center',
					'text-ignore-placement': true
				},
				paint: {
					'text-color': '#263238',
					'text-halo-color': '#fff',
					'text-halo-width': 1
				}
			});

			map.addLayer({
				id: LAYER_SYMBOL,
				type: 'symbol',
				source: SOURCE_SYMBOL,
				layout: {
					'icon-image': 'circle',
					'icon-size': 1
				},
				paint: {
					'icon-color': 'rgb(245,169,71)'
				}
			});
			map.addLayer({
				id: LAYER_SYMBOL_RADIUS,
				type: 'symbol',
				source: SOURCE_SYMBOL_RADIUS,
				layout: {
					'icon-image': 'square',
					'icon-size': 1
				},
				paint: {
					'icon-color': 'rgb(245,169,71)'
				}
			});
			map.on('mousedown', LAYER_SYMBOL_RADIUS, (e) => {
				// Prevent the default map drag behavior.
				e.preventDefault();
				isDragging = true;

				// Set a cursor indicator
				map.getCanvasContainer().style.cursor = 'grab';

				// Mouse events
				map.on('mousemove', mouseMoveOnPoint);
				map.once('mouseup', mouseUpOnPoint);
			});

			map.on('mousedown', LAYER_SYMBOL, (e) => {
				// Prevent the default map drag behavior.
				e.preventDefault();
				isDragging = true;

				// Set a cursor indicator
				map.getCanvasContainer().style.cursor = 'grab';

				// Mouse events
				map.on('mousemove', mouseMoveOnCenter);
				map.once('mouseup', mouseUpOnCenter);
			});
		}
	};

	const updateCircleFeature = () => {
		const feature = getGeoCircle([coordinatesCenter[0], coordinatesRadius[0]]);
		if (feature) {
			if (!map?.getSource(SOURCE_CIRCLE)) {
				map.addSource(SOURCE_CIRCLE, {
					type: 'geojson',
					data: feature
				});
				map.addLayer({
					id: LAYER_CIRCLE,
					type: 'fill',
					source: SOURCE_CIRCLE,
					paint: {
						'fill-outline-color': 'rgb(245,255,0)',
						'fill-color': 'rgb(245,255,0)',
						'fill-opacity': 0.3
					}
				});
				map.addLayer({
					id: LAYER_CIRCLE_OUTLINE,
					type: 'line',
					source: SOURCE_CIRCLE,
					paint: {
						'line-color': 'rgb(245,169,71)',
						'line-width': 3,
						'line-dasharray': [1, 2, 1]
					}
				});
			} else {
				// eslint-disable-next-line @typescript-eslint/ban-ts-comment
				// @ts-ignore
				map.getSource(SOURCE_CIRCLE).setData(feature);
			}
			circleFeatures.update(() => [feature]);
		} else {
			circleFeatures.update(() => []);
		}
	};

	const geoLineString = (coordinates: number[][] = []) => {
		let length = 0;
		if (coordinates.length > 1) {
			length = distance(coordinates[0], coordinates[1], { units: 'meters' });
		}
		// eslint-disable-next-line @typescript-eslint/ban-ts-comment
		// @ts-ignore
		return {
			type: 'Feature',
			properties: {
				text: labelFormat(length),
				length: (length * 1000).toFixed()
			},
			geometry: {
				type: 'LineString',
				coordinates
			}
		};
	};

	const geoPoint = (coordinates: number[][] = []) => {
		let sum = 0;
		return {
			// eslint-disable-next-line @typescript-eslint/ban-ts-comment
			// @ts-ignore
			type: 'FeatureCollection',
			features: coordinates.map((c, i) => {
				if (i > 0) {
					// eslint-disable-next-line @typescript-eslint/ban-ts-comment
					// @ts-ignore
					sum += distance(coordinates[i - 1], coordinates[i], { units });
				}
				return {
					type: 'Feature',
					properties: {
						id: i + 1,
						length: (sum * 1000).toFixed()
					},
					geometry: {
						type: 'Point',
						coordinates: c
					}
				};
			})
		};
	};

	const getGeoCircle = (coordinates: number[][] = []) => {
		if (coordinates.length !== 2) return;
		const center = coordinates[0];
		const radius = distance(center, coordinates[1], { units: 'kilometers' });
		const bearingVal = bearing(center, coordinates[1]);
		const geodesicCoordinates = createGeodesicCircle(center, radius, bearingVal);

		const circlePolygon = {
			type: 'Feature',
			properties: {},
			geometry: {
				type: 'Polygon',
				coordinates: [geodesicCoordinates]
			}
		};
		// eslint-disable-next-line @typescript-eslint/ban-ts-comment
		// @ts-ignore
		const circleArea = area(circlePolygon);
		circlePolygon.properties['area'] = circleArea;
		// eslint-disable-next-line @typescript-eslint/ban-ts-comment
		// @ts-ignore
		return circlePolygon;
	};

	const labelFormat = (length: number) => {
		let lengthLabel = `${length.toFixed(2)} m`;
		if (length < 1) {
			lengthLabel = `${(length / 1000).toFixed()} km`;
		}
		return `${lengthLabel}`;
	};

	const createGeodesicCircle = (center: number[], radius: number, bearing: number, steps = 64) => {
		const coordinates: number[] = [];
		for (let i = 0; i < steps; ++i) {
			coordinates.push(destinationPoint(center, radius, bearing + (360 * -i) / steps));
		}
		coordinates.push(coordinates[0]);

		return coordinates;
	};

	const destinationPoint = (start: number[], distance: number, bearing: number) => {
		// radius used by mapbox-gl, see https://github.com/mapbox/mapbox-gl-js/blob/main/src/geo/lng_lat.js#L11
		const DEFAULT_RADIUS = 6371.0088;
		return geodesyDestinationPoint(start, distance, bearing, DEFAULT_RADIUS);
	};

	const mouseMoveOnPoint = (e: MapMouseEvent) => {
		if (!isDragging) return;
		const coords = e.lngLat;
		isDragging = true;
		map.getCanvasContainer().style.cursor = 'grabbing';
		coordinatesRadius = [[coords.lng, coords.lat]];
		// eslint-disable-next-line @typescript-eslint/ban-ts-comment
		// @ts-ignore
		map.getSource(SOURCE_LINE).setData(geoLineString([coordinatesCenter[0], coordinatesRadius[0]]));
		// eslint-disable-next-line @typescript-eslint/ban-ts-comment
		// @ts-ignore
		map.getSource(SOURCE_SYMBOL_RADIUS).setData(geoPoint(coordinatesRadius));
		updateCircleFeature();
	};

	const mouseUpOnPoint = () => {
		if (!isDragging) return;
		isDragging = false;
		map.getCanvasContainer().style.cursor = '';
		map.off('mousemove', mouseMoveOnPoint);
	};

	const mouseMoveOnCenter = (e: MapMouseEvent) => {
		if (!isDragging) return;
		const coords = e.lngLat;
		isDragging = true;
		map.getCanvasContainer().style.cursor = 'grabbing';
		coordinatesCenter = [[coords.lng, coords.lat]];
		// eslint-disable-next-line @typescript-eslint/ban-ts-comment
		// @ts-ignore
		map.getSource(SOURCE_LINE).setData(geoLineString([coordinatesCenter[0], coordinatesRadius[0]]));
		// eslint-disable-next-line @typescript-eslint/ban-ts-comment
		// @ts-ignore
		map.getSource(SOURCE_SYMBOL).setData(geoPoint(coordinatesCenter));
		updateCircleFeature();
	};

	const mouseUpOnCenter = () => {
		if (!isDragging) return;
		isDragging = false;
		map.getCanvasContainer().style.cursor = '';
		map.off('mousemove', mouseMoveOnCenter);
	};
</script>

<Group variant="raised">
	<Button on:click={() => drawStart()} variant="raised">
		<Label>
			<Icon class="material-icons" style="margin: 0;vertical-align:middle">add_circle</Icon>
			Add circle
		</Label>
	</Button>
	<div use:GroupItem>
		<Button
			on:click={() => menu.setOpen(true)}
			variant="raised"
			style="padding: 0; min-width: 36px;"
		>
			<Icon class="material-icons" style="margin: 0;">arrow_drop_down</Icon>
		</Button>
		<Menu bind:this={menu} anchorCorner="TOP_LEFT">
			<List>
				<Item on:SMUI:action={() => deleteFeatures()}>
					<Icon class="material-icons" style="margin: 0;">delete</Icon>
					<Text>Delete circle</Text>
				</Item>
			</List>
		</Menu>
	</div>
</Group>
