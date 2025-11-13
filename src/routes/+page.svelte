<script>
	import * as Handy from '@ohdoki/handy-sdk';
	import { onMount, tick, onDestroy } from 'svelte';
	import VideoItem from '$lib/components/VideoItem.svelte';
	import ScriptItem from '$lib/components/ScriptItem.svelte';
	import ImageItem from '$lib/components/ImageItem.svelte';
	import AudioItem from '$lib/components/AudioItem.svelte';

	const HANDY = Handy.init();

	onMount(() => {
		HANDY.attachUUI();
		/** @type {Array<() => void>} */
		const cleanups = [];
		// initialize theme
		try {
			const stored = localStorage.getItem('theme');
			const mql = window.matchMedia('(prefers-color-scheme: dark)');
			applyTheme(stored ? stored === 'dark' : mql.matches);
			const onChange = /** @param {MediaQueryListEvent} e */ (e) => {
				if (!localStorage.getItem('theme')) applyTheme(e.matches);
			};
			mql.addEventListener?.('change', onChange);
			cleanups.push(() => mql.removeEventListener?.('change', onChange));
		} catch (e) {}
		// fullscreen state watcher
		const onFs = () => {
			isFullscreen = !!document.fullscreenElement || isPseudoFs;
		};
		document.addEventListener('fullscreenchange', onFs);
		cleanups.push(() => document.removeEventListener('fullscreenchange', onFs));
		return () => {
			for (const fn of cleanups)
				try {
					fn();
				} catch {}
		};
	});

	let isDark = false;
	/** @param {boolean} dark */
	function applyTheme(dark) {
		isDark = !!dark;
		document.documentElement.classList.toggle('dark', isDark);
		try {
			localStorage.setItem('theme', isDark ? 'dark' : 'light');
			const meta = document.querySelector('meta[name="theme-color"]');
			if (meta) meta.setAttribute('content', isDark ? '#0f172a' : '#f3f4f6');
		} catch {}
	}
	function toggleTheme() {
		applyTheme(!isDark);
	}

	/** @typedef {{ file: File }} VideoEntry */
	/** @typedef {{ file: File }} AudioEntry */
	/** @typedef {{ file: File }} ScriptEntry */
	/** @typedef {{ file: File }} ImageEntry */

	/** @type {VideoEntry[]} */
	let videos = [];
	/** @type {AudioEntry[]} */
	let audios = [];
	/** @type {ImageEntry[]} */
	let images = [];
	/** @type {ScriptEntry[]} */
	let scripts = [];

	/** @type {HTMLInputElement|null} */
	let filePickerEl = null;
	/** @type {HTMLInputElement|null} */
	let importConfigEl = null;
	function openFilePicker() {
		filePickerEl?.click();
	}

	// ---- Import/Export arrangements & keybinds ----
	const fileKeyOf = /** @param {File} f */ (f) => `${f.name}:${f.size}`;

	function exportConfig() {
		/**
		 * @typedef {{
		 *  version: number,
		 *  arrangement: any[],
		 *  keybinds: any[]
		 * }} ExportPayload
		 */
		/** @type {ExportPayload} */
		const payload = {
			version: 1,
			arrangement: arrangement
				.map((seg) => {
					if (seg.kind === 'media') {
						return {
							kind: 'media',
							mediaType: seg.mediaType,
							fileKey: fileKeyOf(seg.file),
							start: seg.start ?? 0,
							end: seg.end ?? 0,
							duration: seg.duration ?? 0,
							muted: !!seg.muted,
							pairedScriptKey: seg.pairedScript ? fileKeyOf(seg.pairedScript) : undefined,
							pairedAudioKey: seg.pairedAudio ? fileKeyOf(seg.pairedAudio) : undefined,
							pairedAudioStart: seg.pairedAudioStart ?? 0,
							pairedAudioEnd: seg.pairedAudioEnd ?? 0
						};
					}
					if (seg.kind === 'script') {
						return { kind: 'script', fileKey: fileKeyOf(seg.file) };
					}
					if (seg.kind === 'jump') {
						return { kind: 'jump', targetIndex: seg.targetIndex, checkExpr: seg.checkExpr || '' };
					}
					if (seg.kind === 'repeat') {
						return {
							kind: 'repeat',
							times: seg.times,
							fromIndex: seg.fromIndex,
							checkExpr: seg.checkExpr || ''
						};
					}
					if (seg.kind === 'js') {
						return { kind: 'js', code: seg.code || '' };
					}
					if (seg.kind === 'pause') {
						return { kind: 'pause', seconds: seg.seconds || 0, checkExpr: seg.checkExpr || '' };
					}
					if (seg.kind === 'custom') {
						return {
							kind: 'custom',
							speed: seg.speed,
							length: seg.length,
							speedInc: seg.speedInc,
							lengthInc: seg.lengthInc,
							randomize: !!seg.randomize,
							randomFreq: seg.randomFreq,
							freqVar: seg.freqVar,
							speedVar: seg.speedVar,
							strokeVar: seg.strokeVar
						};
					}
					return null;
				})
				.filter(Boolean),
			keybinds: keybinds.map((kb) => ({
				combo: kb.combo,
				actions: kb.actions.map((act) => ({ ...act }))
			}))
		};
		const blob = new Blob([JSON.stringify(payload, null, 2)], { type: 'application/json' });
		const url = URL.createObjectURL(blob);
		const a = document.createElement('a');
		a.href = url;
		a.download = 'handy-companion-config.json';
		document.body.appendChild(a);
		a.click();
		setTimeout(() => {
			URL.revokeObjectURL(url);
			document.body.removeChild(a);
		}, 0);
	}

	/** @param {Event & { currentTarget: HTMLInputElement }} e */
	async function handleImportConfigChange(e) {
		const file = e.currentTarget.files?.[0];
		e.currentTarget.value = '';
		if (!file) return;
		try {
			const text = await file.text();
			const payload = JSON.parse(text);
			if (!payload || typeof payload !== 'object') throw new Error('Invalid file');
			const ver = Number(payload.version ?? 1);
			if (ver < 1) throw new Error('Unsupported version');

			// Build file maps to resolve references
			const vMap = new Map(videos.map((v) => [fileKeyOf(v.file), v.file]));
			const aMap = new Map(audios.map((a) => [fileKeyOf(a.file), a.file]));
			const iMap = new Map(images.map((im) => [fileKeyOf(im.file), im.file]));
			const sMap = new Map(scripts.map((s) => [fileKeyOf(s.file), s.file]));

			/** @type {ArrangementSegment[]} */
			const newArrangement = [];
			let skipped = 0;
			for (const item of payload.arrangement || []) {
				if (!item || typeof item !== 'object' || !item.kind) continue;
				if (item.kind === 'media') {
					/** @type {'video'|'audio'|'image'} */
					const mt = item.mediaType;
					let fileRef = null;
					if (mt === 'video') fileRef = vMap.get(item.fileKey);
					else if (mt === 'audio') fileRef = aMap.get(item.fileKey);
					else if (mt === 'image') fileRef = iMap.get(item.fileKey);
					if (!fileRef) {
						skipped++;
						continue;
					}
					/** @type {ArrangementMediaSegment} */
					const seg = {
						id: genId(),
						kind: 'media',
						mediaType: mt,
						file: fileRef,
						start: Number(item.start || 0) || 0,
						end: Number(item.end || 0) || 0,
						duration: Number(item.duration || 0) || 0,
						muted: !!item.muted,
						pairedScript: undefined,
						pairedAudio: undefined,
						pairedAudioStart: Number(item.pairedAudioStart || 0) || 0,
						pairedAudioEnd: Number(item.pairedAudioEnd || 0) || 0
					};
					if (item.pairedScriptKey) seg.pairedScript = sMap.get(item.pairedScriptKey) || null;
					if (item.pairedAudioKey) seg.pairedAudio = aMap.get(item.pairedAudioKey) || null;
					newArrangement.push(seg);
					continue;
				}
				if (item.kind === 'script') {
					const f = sMap.get(item.fileKey);
					if (!f) {
						skipped++;
						continue;
					}
					newArrangement.push({ id: genId(), kind: 'script', file: f });
					continue;
				}
				if (item.kind === 'jump') {
					newArrangement.push({
						id: genId(),
						kind: 'jump',
						targetIndex: Math.max(1, Number(item.targetIndex || 1)),
						checkExpr: (item.checkExpr || '').trim()
					});
					continue;
				}
				if (item.kind === 'repeat') {
					newArrangement.push({
						id: genId(),
						kind: 'repeat',
						times: Math.max(1, Number(item.times || 1)),
						fromIndex: Math.max(1, Number(item.fromIndex || 1)),
						checkExpr: (item.checkExpr || '').trim()
					});
					continue;
				}
				if (item.kind === 'js') {
					newArrangement.push({ id: genId(), kind: 'js', code: String(item.code || '') });
					continue;
				}
				if (item.kind === 'pause') {
					newArrangement.push({
						id: genId(),
						kind: 'pause',
						seconds: Math.max(0, Number(item.seconds || 0)),
						checkExpr: (item.checkExpr || '').trim()
					});
					continue;
				}
				if (item.kind === 'custom') {
					newArrangement.push({
						id: genId(),
						kind: 'custom',
						speed: Number(item.speed || 0),
						length: Number(item.length || 0),
						speedInc: Number(item.speedInc || 0),
						lengthInc: Number(item.lengthInc || 0),
						randomize: !!item.randomize,
						randomFreq: Number(item.randomFreq || 0),
						freqVar: Number(item.freqVar || 0),
						speedVar: Number(item.speedVar || 0),
						strokeVar: Number(item.strokeVar || 0)
					});
					continue;
				}
			}
			arrangement = newArrangement;

			/** @type {Keybind[]} */
			const newKeybinds = [];
			for (const kb of payload.keybinds || []) {
				if (!kb || typeof kb !== 'object') continue;
				const combo = String(kb.combo || '').trim();
				if (!combo) continue;
				const actions = [];
				for (const a of kb.actions || []) {
					if (!a || typeof a !== 'object' || !a.type) continue;
					if (a.type === 'js')
						actions.push({
							type: 'js',
							code: String(a.code || ''),
							checkExpr: (a.checkExpr || '').trim()
						});
					else if (a.type === 'goto')
						actions.push({
							type: 'goto',
							random: !!a.random,
							targetIndex: Math.max(1, Number(a.targetIndex || 1)),
							checkExpr: (a.checkExpr || '').trim()
						});
					else if (a.type === 'skip')
						actions.push({
							type: 'skip',
							amount: Math.max(1, Number(a.amount || 1)),
							checkExpr: (a.checkExpr || '').trim()
						});
					else if (a.type === 'pause')
						actions.push({
							type: 'pause',
							seconds: Math.max(0, Number(a.seconds || 0)),
							checkExpr: (a.checkExpr || '').trim()
						});
				}
				newKeybinds.push({ id: genId(), combo, actions });
			}
			keybinds = newKeybinds;

			const msg =
				`Imported ${newArrangement.length} segment(s) and ${newKeybinds.length} keybind(s)` +
				(skipped ? ` (skipped ${skipped} unresolved item(s))` : '');
			console.info(msg);
			try {
				alert(msg);
			} catch {}
		} catch (err) {
			console.error('Import failed', err);
			try {
				alert('Import failed: ' + (err?.message || err));
			} catch {}
		}
	}

	// Arrangement types
	/** @typedef {('video'|'audio'|'image')} MediaType */
	/** @typedef {{ id:string, kind:'media', mediaType:MediaType, file: File, start?: number, end?: number, duration?: number, pairedScript?: File|null, muted?: boolean, pairedAudio?: File|null, pairedAudioStart?: number, pairedAudioEnd?: number }} ArrangementMediaSegment */
	/** @typedef {{ id:string, kind:'script', file: File }} ArrangementScriptSegment */
	/** @typedef {{ id:string, kind:'jump', targetIndex: number, checkExpr?: string }} ArrangementJumpSegment */
	/** @typedef {{ id:string, kind:'repeat', times: number, fromIndex: number, checkExpr?: string }} ArrangementRepeatSegment */
	/** @typedef {{ id:string, kind:'js', code: string }} ArrangementSetVarSegment */
	/** @typedef {{ id:string, kind:'pause', seconds:number, checkExpr?: string }} ArrangementPauseSegment */
	/** @typedef {{ id:string, kind:'custom', speed:number, length:number, speedInc:number, lengthInc:number, randomize:boolean, randomFreq:number, freqVar:number, speedVar:number, strokeVar:number }} ArrangementCustomSegment */
	/** @typedef {(ArrangementMediaSegment|ArrangementScriptSegment|ArrangementJumpSegment|ArrangementRepeatSegment|ArrangementSetVarSegment|ArrangementPauseSegment|ArrangementCustomSegment)} ArrangementSegment */

	/** @type {ArrangementSegment[]} */
	let arrangement = [];

	// Keybind types
	/** @typedef {{ type:'js', code:string, checkExpr?: string } | { type:'goto', random?: boolean, targetIndex?: number, checkExpr?: string } | { type:'skip', amount:number, checkExpr?: string } | { type:'pause', seconds:number, checkExpr?: string }} KeyAction */
	/** @typedef {{ id:string, combo:string, actions: KeyAction[] }} Keybind */
	/** @type {Keybind[]} */
	let keybinds = [];

	let recordingKey = false;
	let recordedCombo = '';

	/** @param {KeyboardEvent} e */
	function formatKeyCombo(e) {
		const parts = [];
		if (e.ctrlKey) parts.push('Ctrl');
		if (e.shiftKey) parts.push('Shift');
		if (e.altKey) parts.push('Alt');
		if (e.metaKey) parts.push('Meta');
		let key = e.key;
		if (key === ' ') key = 'Space';
		if (key.length === 1) key = key.toUpperCase();
		const skip = ['Control', 'Shift', 'Alt', 'Meta'];
		if (!skip.includes(key)) parts.push(key);
		return parts.join('+');
	}

	function startRecordingKey() {
		recordedCombo = '';
		recordingKey = true;
		const handler = /** @param {KeyboardEvent} e */ (e) => {
			e.preventDefault();
			e.stopPropagation();
			recordedCombo = formatKeyCombo(e);
			recordingKey = false;
			window.removeEventListener('keydown', handler, true);
			addKeybindFromRecorded();
		};
		window.addEventListener('keydown', handler, true);
	}

	function addKeybindFromRecorded() {
		const combo = (recordedCombo || '').trim();
		if (!combo) return;
		keybinds = [...keybinds, { id: genId(), combo, actions: [] }];
		recordedCombo = '';
	}

	/** @param {number} kbIdx */
	function removeKeybind(kbIdx) {
		keybinds = keybinds.filter((_, i) => i !== kbIdx);
	}
	/** @param {number} kbIdx */
	function addAction(kbIdx) {
		const kb = keybinds[kbIdx];
		if (!kb) return;
		kb.actions.push({ type: 'js', code: '' });
		keybinds = keybinds; // trigger
	}
	/** @param {number} kbIdx @param {number} aIdx */
	function removeAction(kbIdx, aIdx) {
		const kb = keybinds[kbIdx];
		if (!kb) return;
		kb.actions.splice(aIdx, 1);
		keybinds = keybinds;
	}
	/** @param {number} kbIdx @param {number} aIdx @param {'js'|'goto'|'skip'|'pause'} newType */
	function changeActionType(kbIdx, aIdx, newType) {
		const kb = keybinds[kbIdx];
		if (!kb) return;
		/** @type {KeyAction} */
		let next;
		if (newType === 'js') next = /** @type {KeyAction} */ ({ type: 'js', code: '', checkExpr: '' });
		else if (newType === 'goto')
			next = /** @type {KeyAction} */ ({
				type: 'goto',
				random: false,
				targetIndex: 1,
				checkExpr: ''
			});
		else if (newType === 'skip')
			next = /** @type {KeyAction} */ ({ type: 'skip', amount: 1, checkExpr: '' });
		else next = /** @type {KeyAction} */ ({ type: 'pause', seconds: 1, checkExpr: '' });
		kb.actions[aIdx] = next;
		keybinds = keybinds;
	}

	const genId = () =>
		crypto?.randomUUID
			? crypto.randomUUID()
			: `${Date.now()}-${Math.random().toString(36).slice(2, 8)}`;
	/** @param {string} name */
	const baseName = (name) => (name.includes('.') ? name.slice(0, name.lastIndexOf('.')) : name);

	// Remove helpers per list
	/** @param {VideoEntry} item */
	function removeVideo(item) {
		videos = videos.filter((it) => it !== item);
	}
	/** @param {AudioEntry} item */
	function removeAudio(item) {
		audios = audios.filter((it) => it !== item);
	}
	/** @param {ImageEntry} item */
	function removeImage(item) {
		images = images.filter((it) => it !== item);
	}
	/** @param {ScriptEntry} item */
	function removeScript(item) {
		scripts = scripts.filter((it) => it !== item);
	}

	// (pairing removed)

	/**
	 * @param {Event & { currentTarget: HTMLInputElement }} e
	 */
	function handleFilesChange(e) {
		const files = Array.from(e.currentTarget.files || []);

		/** @type {VideoEntry[]} */
		const newVideos = [];
		/** @type {AudioEntry[]} */
		const newAudios = [];
		/** @type {ScriptEntry[]} */
		const newScripts = [];
		/** @type {ImageEntry[]} */
		const newImages = [];

		for (const f of files) {
			const name = (f.name || '').toLowerCase();
			const type = f.type || '';
			/** @param {string} p */
			const byType = (p) => type.startsWith(p);
			/** @param {string[]} exts */
			const byExt = (...exts) => exts.some((ext) => name.endsWith(ext));
			if (byExt('.json')) {
				handleImportConfigChange({
					currentTarget: { files: [f] }
				});
			} else if (
				byType('video/') ||
				byExt('.mp4', '.webm', '.mkv', '.mov', '.avi', '.m4v', '.wmv', '.3gp', '.ts')
			) {
				newVideos.push({ file: f });
			} else if (
				byType('audio/') ||
				byExt('.mp3', '.wav', '.flac', '.ogg', '.m4a', '.aac', '.opus', '.wma', '.aiff', '.alac')
			) {
				newAudios.push({ file: f });
			} else if (
				byType('image/') ||
				byExt('.jpg', '.jpeg', '.png', '.gif', '.webp', '.bmp', '.svg', '.avif')
			) {
				newImages.push({ file: f });
			} else if (byExt('.funscript')) {
				newScripts.push({ file: f });
			}
		}

		// Append with basic de-duplication by name+size
		/** @param {{file: File}[]} list @param {{file: File}[]} adds */
		const dedupAppend = (list, adds) => {
			const existing = new Set(list.map((it) => `${it.file.name}:${it.file.size}`));
			for (const item of adds) {
				const key = `${item.file.name}:${item.file.size}`;
				if (!existing.has(key)) {
					list.push(item);
					existing.add(key);
				}
			}
		};

		dedupAppend(videos, newVideos);
		dedupAppend(audios, newAudios);
		dedupAppend(images, newImages);
		dedupAppend(scripts, newScripts);

		// Reassign arrays to trigger Svelte reactivity after in-place mutations
		videos = videos;
		audios = audios;
		images = images;
		scripts = scripts;
		// reset input to allow re-adding the same files if needed
		e.currentTarget.value = '';
	}

	// Arrangement builder state
	/** @type {'video'|'audio'|'image'|'script'|'jump'|'repeat'|'js'|'pause'|'custom'} */
	let addType = 'video';
	let addMediaIndex = -1; // index within the selected media list
	let addScriptIndex = -1; // index within scripts
	let addJumpTarget = 1; // 1-based index
	let addRepeatTimes = 2;
	let addSetCode = '';
	let addPauseSeconds = 1;
	let addPauseCheck = '';

	// initial fields for Custom movement
	let addCustomSpeed = 50;
	let addCustomLength = 50;
	let addCustomSpeedInc = 0;
	let addCustomLengthInc = 0;
	let addCustomRandomize = false;
	let addCustomRandomFreq = 1; // Hz
	let addCustomFreqVar = 0.25; // 0..1
	let addCustomSpeedVar = 0.2; // 0..1
	let addCustomStrokeVar = 0.2; // 0..1

	function addSegment() {
		if (addType === 'video') {
			if (addMediaIndex >= 0 && addMediaIndex < videos.length) {
				arrangement = [
					...arrangement,
					{
						id: genId(),
						kind: 'media',
						mediaType: 'video',
						file: videos[addMediaIndex].file,
						start: 0,
						end: 0,
						muted: false,
						pairedScript: null
					}
				];
			}
		} else if (addType === 'audio') {
			if (addMediaIndex >= 0 && addMediaIndex < audios.length) {
				arrangement = [
					...arrangement,
					{
						id: genId(),
						kind: 'media',
						mediaType: 'audio',
						file: audios[addMediaIndex].file,
						start: 0,
						end: 0,
						pairedScript: null
					}
				];
			}
		} else if (addType === 'image') {
			if (addMediaIndex >= 0 && addMediaIndex < images.length) {
				arrangement = [
					...arrangement,
					{
						id: genId(),
						kind: 'media',
						mediaType: 'image',
						file: images[addMediaIndex].file,
						duration: 5,
						pairedScript: null
					}
				];
			}
		} else if (addType === 'script') {
			if (addScriptIndex >= 0 && addScriptIndex < scripts.length) {
				arrangement = [
					...arrangement,
					{ id: genId(), kind: 'script', file: scripts[addScriptIndex].file }
				];
			}
		} else if (addType === 'jump') {
			const target = Math.max(1, Math.min(addJumpTarget | 0, arrangement.length + 1));
			arrangement = [...arrangement, { id: genId(), kind: 'jump', targetIndex: target }];
		} else if (addType === 'repeat') {
			const times = Math.max(1, addRepeatTimes | 0);
			// default to repeat starting from the previous segment (1-based index), or 1 if empty
			const fromIndex = Math.max(1, arrangement.length);
			arrangement = [...arrangement, { id: genId(), kind: 'repeat', times, fromIndex }];
		} else if (addType === 'js') {
			const code = (addSetCode || '').trim();
			arrangement = [...arrangement, { id: genId(), kind: 'js', code }];
			addSetCode = '';
		} else if (addType === 'pause') {
			const seconds = Math.max(0, addPauseSeconds | 0);
			arrangement = [
				...arrangement,
				{ id: genId(), kind: 'pause', seconds, checkExpr: (addPauseCheck || '').trim() }
			];
			addPauseSeconds = 1;
			addPauseCheck = '';
		} else if (addType === 'custom') {
			arrangement = [
				...arrangement,
				{
					id: genId(),
					kind: 'custom',
					speed: +addCustomSpeed,
					length: +addCustomLength,
					speedInc: +addCustomSpeedInc,
					lengthInc: +addCustomLengthInc,
					randomize: !!addCustomRandomize,
					randomFreq: +addCustomRandomFreq,
					freqVar: +addCustomFreqVar,
					speedVar: +addCustomSpeedVar,
					strokeVar: +addCustomStrokeVar
				}
			];
		}
	}

	/** @param {number} idx */
	function moveUp(idx) {
		if (idx <= 0) return;
		const copy = arrangement.slice();
		[copy[idx - 1], copy[idx]] = [copy[idx], copy[idx - 1]];
		arrangement = copy;
	}
	/** @param {number} idx */
	function moveDown(idx) {
		if (idx >= arrangement.length - 1) return;
		const copy = arrangement.slice();
		[copy[idx + 1], copy[idx]] = [copy[idx], copy[idx + 1]];
		arrangement = copy;
	}
	/** @param {number} idx */
	function removeSegment(idx) {
		arrangement = arrangement.filter((_, i) => i !== idx);
	}
	/** @param {number} idx @param {number} scriptIndex */
	function pairScriptTo(idx, scriptIndex) {
		const seg = arrangement[idx];
		if (!seg || seg.kind !== 'media') return;
		if (scriptIndex >= 0 && scriptIndex < scripts.length) {
			seg.pairedScript = scripts[scriptIndex].file;
			arrangement = arrangement; // trigger update
		}
	}
	/** @param {number} idx */
	function unpairScript(idx) {
		const seg = arrangement[idx];
		if (seg && seg.kind === 'media') {
			seg.pairedScript = null;
			arrangement = arrangement;
		}
	}

	/** @param {number} idx @param {number} audioIndex */
	function pairAudioTo(idx, audioIndex) {
		const seg = arrangement[idx];
		if (!seg || seg.kind !== 'media') return;
		if (seg.mediaType !== 'video' && seg.mediaType !== 'image') return;
		if (audioIndex >= 0 && audioIndex < audios.length) {
			seg.pairedAudio = audios[audioIndex].file;
			seg.pairedAudioStart = 0;
			seg.pairedAudioEnd = 0; // 0 = full length
			arrangement = arrangement;
		}
	}
	/** @param {number} idx */
	function unpairAudio(idx) {
		const seg = arrangement[idx];
		if (seg && seg.kind === 'media') {
			seg.pairedAudio = null;
			seg.pairedAudioStart = undefined;
			seg.pairedAudioEnd = undefined;
			arrangement = arrangement;
		}
	}
	function autoPairScripts() {
		const map = new Map();
		for (const sc of scripts) {
			map.set(baseName(sc.file.name).toLowerCase(), sc.file);
		}
		let paired = 0;
		for (const seg of arrangement) {
			if (seg.kind === 'media' && !seg.pairedScript) {
				const bn = baseName(seg.file.name).toLowerCase();
				const match = map.get(bn);
				if (match) {
					seg.pairedScript = match;
					paired++;
				}
			}
		}
		arrangement = arrangement;
		// Optionally we could toast the count; for now, silent update.
	}

	// ===== Player state =====
	let playing = false;
	let currentIndex = 0; // 0-based
	/** @type {Record<string, any>} */
	let vars = {};

	// preview state (single-segment preview outside of Play All)
	let previewingIndex = /** @type {number|null} */ (null);

	// current media rendering
	let visualType = /** @type {null|'video'|'image'|'audio'} */ (null);
	let visualUrl = '';
	let visualMuted = false;
	let audioUrl = '';

	/** @type {HTMLVideoElement|null} */
	let videoEl = null;
	/** @type {HTMLAudioElement|null} */
	let audioEl = null;
	/** @type {HTMLDivElement|null} */
	let stageEl = null;
	/** @type {HTMLImageElement|null} */
	let imgEl = null;
	let isFullscreen = false;
	let isPseudoFs = false;

	function getFullscreenTarget() {
		if (visualType === 'video' && videoEl) return videoEl;
		if (visualType === 'image' && imgEl) return imgEl;
		if (stageEl) return stageEl;
		return document.documentElement;
	}

	function toggleFullscreen() {
		const doc = /** @type {any} */ (document);
		const inNative = !!document.fullscreenElement;
		if (inNative || isPseudoFs) {
			if (inNative)
				(document.exitFullscreen || doc.webkitExitFullscreen || doc.msExitFullscreen)?.call(
					document
				);
			if (isPseudoFs) {
				document.documentElement.classList.remove('pseudo-fs');
				isPseudoFs = false;
			}
			isFullscreen = false;
			return;
		}
		// enter: prefer native on stage for video; pseudo for image/audio
		if (visualType === 'video' && stageEl && /** @type {any} */ (stageEl).requestFullscreen) {
			/** @type {any} */ (stageEl).requestFullscreen().catch(() => {
				document.documentElement.classList.add('pseudo-fs');
				isPseudoFs = true;
				isFullscreen = true;
			});
		} else {
			document.documentElement.classList.add('pseudo-fs');
			isPseudoFs = true;
			isFullscreen = true;
		}
	}

	let segmentAbort = { aborted: false };
	let jumpToIndex = /** @type {number|null} */ (null);
	let paused = false;
	/** @type {any} */ let pauseTimer = null;
	/** image timing */
	/** @type {any} */ let imageTimer = null;
	let imageRemainingMs = 0;
	let imageStartAtMs = 0;

	/** @param {string} expr */
	function evalCheck(expr) {
		try {
			// Expose vars and ctx to the expression, and make unknown identifiers resolve to undefined
			// so checks like `a === 1` don't throw when `a` isn't set yet.
			return Function(
				'vars',
				'ctx',
				`const pv = new Proxy(vars, { has: () => true, get: (t, p) => t[p] });
				with (ctx) { with (pv) { return ( ${expr} ); } }`
			)(vars, { index: currentIndex + 1 });
		} catch (e) {
			console.warn('Check eval error', e);
			return false;
		}
	}
	/** @param {string} code */
	function evalCode(code) {
		try {
			Function(
				'vars',
				'ctx',
				`const pv = new Proxy(vars, { has: () => true, get: (t, p) => t[p] });
				with (ctx) { with (pv) { ${code} } }`
			)(vars, { index: currentIndex + 1 });
		} catch (e) {
			console.warn('Code eval error', e);
		}
	}

	function clearUrls() {
		if (visualUrl) URL.revokeObjectURL(visualUrl);
		if (audioUrl) URL.revokeObjectURL(audioUrl);
		visualUrl = '';
		audioUrl = '';
	}

	function cleanupMedia() {
		if (videoEl) {
			videoEl.pause();
			videoEl.src = '';
		}
		if (audioEl) {
			audioEl.pause();
			audioEl.src = '';
		}
		if (imageTimer) {
			clearTimeout(imageTimer);
			imageTimer = null;
		}
		clearUrls();
		visualType = null;
	}

	function stopPlayback() {
		playing = false;
		segmentAbort.aborted = true;
		jumpToIndex = null;
		paused = false;
		if (pauseTimer) {
			clearTimeout(pauseTimer);
			pauseTimer = null;
		}
		cleanupMedia();
		window.removeEventListener('keydown', onPlaybackKeyDown, true);
	}

	function stopPreview() {
		segmentAbort.aborted = true;
		paused = false;
		if (pauseTimer) {
			clearTimeout(pauseTimer);
			pauseTimer = null;
		}
		cleanupMedia();
		previewingIndex = null;
	}

	/** Preview a single segment by index without running the whole playlist. */
	async function previewSegment(idx) {
		const seg = arrangement[idx];
		if (!seg) return;
		// toggle behavior: if already previewing this one, stop
		if (previewingIndex === idx) {
			stopPreview();
			return;
		}
		// stop any ongoing playback or other preview
		if (playing) stopPlayback();
		if (previewingIndex != null) stopPreview();
		segmentAbort = { aborted: false };
		previewingIndex = idx;
		if (seg.kind === 'media') {
			await playMedia(seg);
		}
		// end of preview (if not interrupted)
		if (previewingIndex === idx) {
			stopPreview();
		}
	}

	/** @param {number} seconds */
	function pauseCurrent(seconds) {
		if (seconds <= 0) return;
		if (paused) return;
		paused = true;
		if (videoEl && !videoEl.paused) videoEl.pause();
		if (audioEl && !audioEl.paused) audioEl.pause();
		if (visualType === 'image' && imageTimer) {
			clearTimeout(imageTimer);
			imageTimer = null;
			const elapsed = Math.max(0, Date.now() - imageStartAtMs);
			imageRemainingMs = Math.max(0, imageRemainingMs - elapsed);
		}
		pauseTimer = setTimeout(() => {
			paused = false;
			if (videoEl && videoEl.paused) void videoEl.play().catch(() => {});
			if (audioEl && audioEl.paused) void audioEl.play().catch(() => {});
			if (visualType === 'image' && imageRemainingMs > 0 && !imageTimer) {
				imageTimer = setTimeout(() => {
					imageTimer = null; /* resume end naturally */
				}, imageRemainingMs);
			}
		}, seconds * 1000);
	}

	/** @param {KeyboardEvent} e */
	function onPlaybackKeyDown(e) {
		if (recordingKey) return; // ignore during recording
		const combo = formatKeyCombo(e);
		const hits = keybinds.filter((kb) => kb.combo === combo);
		if (!hits.length) return;
		for (const kb of hits) {
			for (const act of kb.actions) {
				// Guard each action with optional check expression
				if (act.checkExpr && !evalCheck(act.checkExpr)) continue;
				if (act.type === 'js') {
					if (act.code?.trim()) evalCode(act.code);
				} else if (act.type === 'goto') {
					let target = act.random
						? Math.floor(Math.random() * arrangement.length) + 1
						: Math.max(1, Math.min(act.targetIndex || 1, arrangement.length));
					jumpToIndex = target - 1;
					segmentAbort.aborted = true;
					// stop current media immediately
					if (videoEl) videoEl.pause();
					if (audioEl) audioEl.pause();
					if (imageTimer) {
						clearTimeout(imageTimer);
						imageTimer = null;
					}
				} else if (act.type === 'skip') {
					// allow stacking: if a jump is already pending, use it as the base
					const raw = Number(act.amount ?? 1);
					const amt = Number.isFinite(raw) ? raw : 1;
					const base = jumpToIndex != null ? jumpToIndex : currentIndex;
					const target0 = Math.max(0, Math.min(base + amt, arrangement.length - 1));
					jumpToIndex = target0;
					segmentAbort.aborted = true;
					if (videoEl) videoEl.pause();
					if (audioEl) audioEl.pause();
					if (imageTimer) {
						clearTimeout(imageTimer);
						imageTimer = null;
					}
				} else if (act.type === 'pause') {
					pauseCurrent(Math.max(0, act.seconds || 0));
				}
			}
		}
		e.preventDefault();
		e.stopPropagation();
	}

	async function playAll() {
		if (playing) return;
		playing = true;
		currentIndex = 0;
		vars = {};
		segmentAbort = { aborted: false };
		jumpToIndex = null;
		window.addEventListener('keydown', onPlaybackKeyDown, true);

		// runtime state for repeat segments
		const repeatState = new Map(); // seg.id -> remaining count

		while (playing && currentIndex >= 0 && currentIndex < arrangement.length) {
			segmentAbort.aborted = false;
			const seg = arrangement[currentIndex];
			if (seg.kind === 'js') {
				if (seg.code?.trim()) evalCode(seg.code);
				currentIndex++;
				continue;
			}
			if (seg.kind === 'pause') {
				const ok = !seg.checkExpr || evalCheck(seg.checkExpr);
				if (!ok) {
					currentIndex++;
					continue;
				}
				const secs = Math.max(0, seg.seconds || 0);
				if (secs > 0) {
					pauseCurrent(secs);
					await new Promise((resolve) => {
						function spin() {
							if (segmentAbort.aborted || !playing || !paused) return resolve(null);
							setTimeout(spin, 50);
						}
						spin();
					});
				}
				currentIndex++;
				continue;
			}
			if (seg.kind === 'jump') {
				if (!seg.checkExpr || evalCheck(seg.checkExpr)) {
					currentIndex = Math.max(0, Math.min((seg.targetIndex | 0) - 1, arrangement.length - 1));
				} else {
					currentIndex++;
				}
				continue;
			}
			if (seg.kind === 'repeat') {
				const checkOk = !seg.checkExpr || evalCheck(seg.checkExpr);
				if (!checkOk) {
					currentIndex++;
					continue;
				}
				let remaining = repeatState.get(seg.id);
				if (remaining == null) remaining = (seg.times | 0) - 1;
				else remaining = remaining - 1;
				if (remaining >= 0) {
					repeatState.set(seg.id, remaining);
					currentIndex = Math.max(0, Math.min((seg.fromIndex | 0) - 1, arrangement.length - 1));
				} else {
					repeatState.delete(seg.id);
					currentIndex++;
				}
				continue;
			}
			if (seg.kind === 'script') {
				console.log('Executing script:', seg.id);
				// TODO: Integrate Handy script playback
				currentIndex++;
				continue;
			}
			if (seg.kind === 'media') {
				await playMedia(seg);
				if (!playing) break;
				if (jumpToIndex != null) {
					currentIndex = jumpToIndex;
					jumpToIndex = null;
				} else {
					currentIndex++;
				}
				continue;
			}
		}

		stopPlayback();
	}

	/** @param {ArrangementMediaSegment} seg */
	function setupVisual(seg) {
		clearUrls();
		if (seg.mediaType === 'video') {
			visualType = 'video';
			visualMuted = !!seg.muted;
			visualUrl = URL.createObjectURL(seg.file);
			// for video, default to no separate audio unless paired
			audioUrl = '';
		} else if (seg.mediaType === 'image') {
			visualType = 'image';
			visualMuted = false;
			visualUrl = URL.createObjectURL(seg.file);
			// for image, default to no audio unless paired
			audioUrl = '';
		} else {
			// audio-only segment
			visualType = null;
			audioUrl = URL.createObjectURL(seg.file);
		}
		// If paired audio is provided (valid for video/image), override audio track
		if (seg.pairedAudio) {
			audioUrl = URL.createObjectURL(seg.pairedAudio);
		}
	}

	/** @param {ArrangementMediaSegment} seg */
	async function playMedia(seg) {
		setupVisual(seg);
		await tick(); // ensure elements are mounted
		// image
		if (visualType === 'image') {
			const durSec = Math.max(0, seg.duration || 0);
			imageRemainingMs = Math.floor(durSec * 1000);
			if (imageRemainingMs <= 0) return;
			imageStartAtMs = Date.now();
			await tick();
			const a = audioEl && seg.pairedAudio ? audioEl : null;
			await new Promise((resolve) => {
				let settled = false;
				// optional paired audio loop while image is shown
				const s = Math.max(0, seg.pairedAudioStart || 0);
				const aEnd = seg.pairedAudioEnd || 0; // 0 = full clip
				const onAMeta = () => {
					if (!a) return;
					a.removeEventListener('loadedmetadata', onAMeta);
					a.currentTime = s;
					void a.play().catch(() => {});
				};
				const onATime = () => {
					if (!a) return;
					if (aEnd > 0 && a.currentTime >= aEnd - 0.01) {
						a.currentTime = s;
						void a.play().catch(() => {});
					}
				};
				const onAEnded = () => {
					if (!a) return;
					// only loop on natural end when no explicit end provided
					if (aEnd === 0) {
						a.currentTime = s;
						void a.play().catch(() => {});
					}
				};
				function cleanupAudio() {
					if (!a) return;
					a.removeEventListener('loadedmetadata', onAMeta);
					a.removeEventListener('timeupdate', onATime);
					a.removeEventListener('ended', onAEnded);
					a.pause();
				}
				function finish() {
					if (settled) return;
					settled = true;
					if (imageTimer) {
						clearTimeout(imageTimer);
						imageTimer = null;
					}
					cleanupAudio();
					resolve(null);
				}
				// start paired audio if present
				if (a) {
					a.addEventListener('loadedmetadata', onAMeta);
					a.addEventListener('timeupdate', onATime);
					a.addEventListener('ended', onAEnded);
					// ensure src set
					a.src = audioUrl;
				}

				imageTimer = setTimeout(() => {
					finish();
				}, imageRemainingMs);

				// actively watch for aborts (e.g., via skip keybind) and finish early
				(function spin() {
					if (segmentAbort.aborted) return finish();
					setTimeout(spin, 50);
				})();
			});
			return;
		}
		// audio-only segment
		if (seg.mediaType === 'audio') {
			await tick();
			if (!audioEl) return;
			const a = audioEl;
			await new Promise((resolve) => {
				const onMeta = () => {
					a.removeEventListener('loadedmetadata', onMeta);
					if (seg.start) a.currentTime = Math.max(0, seg.start || 0);
					void a.play().catch(() => {});
				};
				const onEnded = () => {
					cleanup();
					resolve(null);
				};
				const onTime = () => {
					if (segmentAbort.aborted) {
						cleanup();
						resolve(null);
						return;
					}
					const end = seg.end || 0;
					if (end > 0 && a.currentTime >= end) {
						cleanup();
						resolve(null);
					}
				};
				function cleanup() {
					a.removeEventListener('loadedmetadata', onMeta);
					a.removeEventListener('ended', onEnded);
					a.removeEventListener('timeupdate', onTime);
				}
				a.addEventListener('loadedmetadata', onMeta);
				a.addEventListener('ended', onEnded);
				a.addEventListener('timeupdate', onTime);
				// ensure src set: for audio-only we set audioUrl in setupVisual; use it
				a.src = audioUrl || URL.createObjectURL(seg.file);
			});
			return;
		}
		// video (optionally with paired audio)
		await tick();
		if (!videoEl) return;
		const v = videoEl;
		const a = audioEl && seg.pairedAudio ? audioEl : null;
		await new Promise((resolve) => {
			const onVMeta = () => {
				v.removeEventListener('loadedmetadata', onVMeta);
				if (seg.start) v.currentTime = Math.max(0, seg.start || 0);
				void v.play().catch(() => {});
				if (a) {
					const s = Math.max(0, seg.pairedAudioStart || 0);
					a.currentTime = s;
					void a.play().catch(() => {});
				}
			};
			const onVEnded = () => {
				cleanup();
				resolve(null);
			};
			const onVTime = () => {
				if (segmentAbort.aborted) {
					cleanup();
					resolve(null);
					return;
				}
				const end = seg.end || 0;
				if (end > 0 && v.currentTime >= end) {
					cleanup();
					resolve(null);
				}
				if (a) {
					const s = Math.max(0, seg.pairedAudioStart || 0);
					const aEnd = seg.pairedAudioEnd || 0;
					if (aEnd > 0 && a.currentTime >= aEnd - 0.01) {
						a.currentTime = s;
						void a.play().catch(() => {});
					}
				}
			};
			const onAEnded = () => {
				if (!a) return;
				// only loop on natural end when no explicit end provided
				if ((seg.pairedAudioEnd || 0) === 0) {
					const s = Math.max(0, seg.pairedAudioStart || 0);
					a.currentTime = s;
					void a.play().catch(() => {});
				}
			};
			function cleanup() {
				v.removeEventListener('loadedmetadata', onVMeta);
				v.removeEventListener('ended', onVEnded);
				v.removeEventListener('timeupdate', onVTime);
				if (a) {
					a.removeEventListener('ended', onAEnded);
					a.pause();
				}
			}
			v.addEventListener('loadedmetadata', onVMeta);
			v.addEventListener('ended', onVEnded);
			v.addEventListener('timeupdate', onVTime);
			// set sources (visualUrl already set in setupVisual)
			v.src = visualUrl;
			if (a) {
				a.addEventListener('ended', onAEnded);
				a.src = audioUrl;
			}
		});
	}
</script>

<header class="app-header">
	<h1 class="grow">Handy Companion</h1>
	<div class="row">
		<button type="button" class="secondary" on:click={exportConfig}>Export config</button>
		<button type="button" class="secondary" on:click={() => importConfigEl?.click()}
			>Import config</button
		>
		<button type="button" class="secondary" on:click={toggleTheme}
			>Switch {isDark ? 'light' : 'dark'}mode</button
		>
	</div>
</header>

<main class="app-main">
	<ol>
		<li>
			<div id="handy-ui"></div>
		</li>
		<li>
			<h3>Add files</h3>
			<div class="row">
				<button type="button" class="secondary" on:click={openFilePicker}>Add files...</button>
				<input
					style="display: none"
					type="file"
					multiple
					accept="video/*,audio/*,image/*,.funscript,.json"
					on:change={handleFilesChange}
					bind:this={filePickerEl}
				/>

				<input
					style="display: none"
					type="file"
					accept="application/json"
					on:change={handleImportConfigChange}
					bind:this={importConfigEl}
				/>
			</div>
			{#if videos.length || audios.length || images.length || scripts.length}
				<div class="media-lists">
					{#if videos.length}
						<div>
							<h4>Videos</h4>
							<ul class="video-list">
								{#each videos as v (`${v.file.name}:${v.file.size}`)}
									<li>
										<VideoItem file={v.file} on:remove={() => removeVideo(v)} />
									</li>
								{/each}
							</ul>
						</div>
					{/if}

					{#if audios.length}
						<div>
							<h4>Audios</h4>
							<ul class="audio-list">
								{#each audios as a (`${a.file.name}:${a.file.size}`)}
									<li>
										<AudioItem file={a.file} on:remove={() => removeAudio(a)} />
									</li>
								{/each}
							</ul>
						</div>
					{/if}

					{#if images.length}
						<div>
							<h4>Images</h4>
							<ul class="image-list">
								{#each images as im (`${im.file.name}:${im.file.size}`)}
									<li>
										<ImageItem file={im.file} on:remove={() => removeImage(im)} />
									</li>
								{/each}
							</ul>
						</div>
					{/if}

					{#if scripts.length}
						<div>
							<h4>Scripts</h4>
							<ul class="script-list">
								{#each scripts as s (`${s.file.name}:${s.file.size}`)}
									<li>
										<ScriptItem file={s.file} on:remove={() => removeScript(s)} />
									</li>
								{/each}
							</ul>
						</div>
					{/if}
				</div>
			{/if}
			<h3>Layout</h3>
			<div class="arrange-builder">
				{#if arrangement.length}
					<ol class="arrangement-list">
						{#each arrangement as seg, idx (seg.id)}
							<li class="segment">
								<div class="seg-head">
									<strong>#{idx + 1}</strong>
									<div class="seg-actions">
										{#if seg.kind === 'media'}
											<button
												type="button"
												class="secondary"
												on:click={() => previewSegment(idx)}
												aria-label="Preview segment"
											>
												{previewingIndex === idx ? 'Stop Preview' : 'Preview'}
											</button>
										{:else}
											<button
												type="button"
												class="secondary"
												disabled
												title="Preview available for media segments"
											>
												Preview
											</button>
										{/if}
										<button type="button" on:click={() => moveUp(idx)} aria-label="Move up"
											>▲</button
										>
										<button type="button" on:click={() => moveDown(idx)} aria-label="Move down"
											>▼</button
										>
										<button
											type="button"
											class="danger"
											on:click={() => removeSegment(idx)}
											aria-label="Remove segment">×</button
										>
									</div>
								</div>

								{#if seg.kind === 'media'}
									<div class="seg-body">
										<div class="pill {seg.mediaType}">{seg.mediaType}</div>
										<div class="row">
											<div class="grow ellipsis" title={seg.file.name}>{seg.file.name}</div>
										</div>
										{#if seg.mediaType === 'video' || seg.mediaType === 'audio'}
											<div class="row">
												<label
													>Start (s) <input
														type="number"
														min="0"
														step="0.1"
														bind:value={seg.start}
													/></label
												>
												<label
													>End (s)
													<input
														type="number"
														min="0"
														step="0.1"
														placeholder="0 = full"
														bind:value={seg.end}
													/>
													{#if (seg.end ?? 0) === 0}
														<span class="hint">Note: 0 equals full length</span>
													{/if}
												</label>
												{#if seg.mediaType === 'video'}
													<label>
														Muted
														<input type="checkbox" bind:checked={seg.muted} />
													</label>
												{/if}
											</div>
										{:else if seg.mediaType === 'image'}
											<div class="row">
												<label
													>Duration (s) <input
														type="number"
														min="0.1"
														step="0.1"
														bind:value={seg.duration}
													/></label
												>
											</div>
										{/if}
										{#if seg.mediaType === 'video' || seg.mediaType === 'image'}
											<div class="row pair-row">
												{#if seg.pairedAudio}
													<div class="paired">
														Paired audio: <span class="ellipsis" title={seg.pairedAudio.name}
															>{seg.pairedAudio.name}</span
														>
													</div>
													<button type="button" class="secondary" on:click={() => unpairAudio(idx)}
														>Unpair</button
													>
												{:else}
													<label>
														Pair audio
														<select on:change={(e) => pairAudioTo(idx, +e.currentTarget.value)}>
															<option value={-1}>Select…</option>
															{#each audios as a, ai}
																<option value={ai}>{a.file.name}</option>
															{/each}
														</select>
													</label>
												{/if}
											</div>
											{#if seg.pairedAudio}
												<div class="row">
													<label
														>Paired audio start (s)
														<input
															type="number"
															min="0"
															step="0.1"
															bind:value={seg.pairedAudioStart}
														/>
													</label>
													<label
														>Paired audio end (s)
														<input
															type="number"
															min="0"
															step="0.1"
															placeholder="0 = full"
															bind:value={seg.pairedAudioEnd}
														/>
														{#if (seg.pairedAudioEnd ?? 0) === 0}
															<span class="hint">Note: 0 means full length</span>
														{/if}
													</label>
												</div>
											{/if}
										{/if}
										<div class="row pair-row">
											{#if seg.pairedScript}
												<div class="paired">
													Paired script: <span class="ellipsis" title={seg.pairedScript.name}
														>{seg.pairedScript.name}</span
													>
												</div>
												<button type="button" class="secondary" on:click={() => unpairScript(idx)}
													>Unpair</button
												>
											{:else}
												<label>
													Pair script
													<select on:change={(e) => pairScriptTo(idx, +e.currentTarget.value)}>
														<option value={-1}>Select…</option>
														{#each scripts as s, si}
															<option value={si}>{s.file.name}</option>
														{/each}
													</select>
												</label>
											{/if}
										</div>
									</div>
								{:else if seg.kind === 'script'}
									<div class="seg-body">
										<div class="pill script">script</div>
										<div class="row">
											<div class="grow ellipsis" title={seg.file.name}>{seg.file.name}</div>
										</div>
									</div>
								{:else if seg.kind === 'jump'}
									<div class="seg-body">
										<div class="pill control">jump</div>
										<div class="row">
											<label
												>Target index <input
													type="number"
													min="1"
													bind:value={seg.targetIndex}
												/></label
											>
											<label
												>Guard condition
												<input
													type="text"
													placeholder="e.g. counter === 1 or score > 10"
													bind:value={seg.checkExpr}
												/>
											</label>
										</div>
									</div>
								{:else if seg.kind === 'repeat'}
									<div class="seg-body">
										<div class="pill control">repeat</div>
										<div class="row">
											<label>Times <input type="number" min="1" bind:value={seg.times} /></label>
											<label
												>From index <input
													type="number"
													min="1"
													bind:value={seg.fromIndex}
												/></label
											>
											<label
												>Guard condition
												<input
													type="text"
													placeholder="e.g. i < 3 or flag === true"
													bind:value={seg.checkExpr}
												/>
											</label>
										</div>
										<div class="hint">
											Repeats from the given 1-based index back to here, the specified number of
											times.
										</div>
									</div>
								{:else if seg.kind === 'js'}
									<div class="seg-body">
										<div class="pill control">Execute JS</div>
										<label class="grow">
											<input
												placeholder="e.g. counter = (counter ?? 0) + 1;"
												bind:value={seg.code}
											/>
										</label>
									</div>
								{:else if seg.kind === 'pause'}
									<div class="seg-body">
										<div class="pill control">pause</div>
										<div class="row">
											<label>
												Seconds
												<input type="number" min="0" step="0.1" bind:value={seg.seconds} />
											</label>
											<label class="grow">
												Guard condition
												<input
													type="text"
													placeholder="e.g. flag === true"
													bind:value={seg.checkExpr}
												/>
											</label>
										</div>
									</div>
								{:else if seg.kind === 'custom'}
									<div class="seg-body">
										<div class="pill control">custom movement</div>
										<div class="row">
											<label class="grow">
												Speed: {seg.speed}
												<input type="range" min="0" max="100" step="1" bind:value={seg.speed} />
											</label>
											<label class="grow">
												Length: {seg.length}
												<input type="range" min="0" max="100" step="1" bind:value={seg.length} />
											</label>
										</div>
										<div class="row">
											<label class="grow">
												Speed increment: {seg.speedInc}
												<input
													type="range"
													min="-10"
													max="10"
													step="0.1"
													bind:value={seg.speedInc}
												/>
											</label>
											<label class="grow">
												Length increment: {seg.lengthInc}
												<input
													type="range"
													min="-10"
													max="10"
													step="0.1"
													bind:value={seg.lengthInc}
												/>
											</label>
										</div>
										<div class="row">
											<label>
												Randomize
												<input type="checkbox" bind:checked={seg.randomize} />
											</label>
										</div>
										<div class="row">
											<label class="grow">
												Random frequency (Hz): {seg.randomFreq}
												<input
													type="range"
													min="0"
													max="5"
													step="0.1"
													bind:value={seg.randomFreq}
												/>
											</label>
											<label class="grow">
												Frequency variability: {seg.freqVar}
												<input type="range" min="0" max="1" step="0.01" bind:value={seg.freqVar} />
											</label>
										</div>
										<div class="row">
											<label class="grow">
												Speed variability: {seg.speedVar}
												<input type="range" min="0" max="1" step="0.01" bind:value={seg.speedVar} />
											</label>
											<label class="grow">
												Stroke variability: {seg.strokeVar}
												<input
													type="range"
													min="0"
													max="1"
													step="0.01"
													bind:value={seg.strokeVar}
												/>
											</label>
										</div>
									</div>
								{/if}
							</li>
						{/each}
					</ol>
				{/if}
				<div class="row arrange-add">
					<label>
						Type
						<select bind:value={addType}>
							<option value="video">Video</option>
							<option value="audio">Audio</option>
							<option value="image">Image</option>
							<option value="script">Script</option>
							<option value="jump">Control: Jump To</option>
							<option value="repeat">Control: Repeat</option>
							<option value="js">Control: Execute JS</option>
							<option value="pause">Control: Pause</option>
							<option value="custom">Control: Custom movement</option>
						</select>
					</label>

					{#if addType === 'video'}
						<label>
							Video
							<select bind:value={addMediaIndex}>
								<option value={-1}>Select a video…</option>
								{#each videos as v, i}
									<option value={i}>{v.file.name}</option>
								{/each}
							</select>
						</label>
					{:else if addType === 'audio'}
						<label>
							Audio
							<select bind:value={addMediaIndex}>
								<option value={-1}>Select an audio…</option>
								{#each audios as a, i}
									<option value={i}>{a.file.name}</option>
								{/each}
							</select>
						</label>
					{:else if addType === 'image'}
						<label>
							Image
							<select bind:value={addMediaIndex}>
								<option value={-1}>Select an image…</option>
								{#each images as im, i}
									<option value={i}>{im.file.name}</option>
								{/each}
							</select>
						</label>
					{:else if addType === 'script'}
						<label>
							Script
							<select bind:value={addScriptIndex}>
								<option value={-1}>Select a script…</option>
								{#each scripts as s, i}
									<option value={i}>{s.file.name}</option>
								{/each}
							</select>
						</label>
					{/if}

					<br />
				</div>
				<div>
					<button type="button" class="primary" on:click={addSegment}>Add</button>
					<button
						type="button"
						class="secondary"
						on:click={autoPairScripts}
						title="Pair scripts to media with the same base name">Auto pair all scripts</button
					>
				</div>
			</div>
			<h3>Keybinds</h3>

			<div class="keybinds">
				{#if keybinds.length}
					<ul class="kb-list">
						{#each keybinds as kb, kbi (kb.id)}
							<li class="kb-item">
								<div class="kb-head">
									<strong>{kb.combo}</strong>
									<button type="button" class="danger" on:click={() => removeKeybind(kbi)}>x</button
									>
								</div>
								<div class="kb-actions">
									{#if kb.actions.length}
										{#each kb.actions as act, ai (ai)}
											<div class="kb-action">
												<label>
													Type
													<select
														value={act.type}
														on:change={(e) =>
															changeActionType(kbi, ai, /** @type {any} */ (e.currentTarget).value)}
													>
														<option value="js">Execute JS</option>
														<option value="goto">Jump To</option>
														<option value="skip">Skip</option>
														<option value="pause">Pause</option>
													</select>
												</label>

												{#if act.type === 'js'}
													<label class="grow">
														JS code
														<input
															type="text"
															placeholder="counter = (counter ?? 0) + 1;"
															bind:value={act.code}
														/>
													</label>
													<label class="grow">
														Check expression (optional)
														<input
															type="text"
															placeholder="e.g. score > 10"
															bind:value={act.checkExpr}
														/>
													</label>
												{:else if act.type === 'skip'}
													<label>
														Amount
														<input type="number" min="1" step="1" bind:value={act.amount} />
													</label>
													<label class="grow">
														Check expression (optional)
														<input
															type="text"
															placeholder="e.g. flag === true"
															bind:value={act.checkExpr}
														/>
													</label>
												{:else if act.type === 'goto'}
													<label>
														Random
														<input type="checkbox" bind:checked={act.random} />
													</label>
													<label>
														Target index
														<input
															type="number"
															min="1"
															bind:value={act.targetIndex}
															disabled={act.random}
														/>
													</label>
													<label class="grow">
														Check expression (optional)
														<input
															type="text"
															placeholder="e.g. flag === true"
															bind:value={act.checkExpr}
														/>
													</label>
												{:else if act.type === 'pause'}
													<label>
														Seconds
														<input type="number" min="0" step="0.1" bind:value={act.seconds} />
													</label>
													<label class="grow">
														Check expression (optional)
														<input
															type="text"
															placeholder="e.g. score > 10"
															bind:value={act.checkExpr}
														/>
													</label>
												{/if}

												<button
													type="button"
													class="secondary"
													on:click={() => removeAction(kbi, ai)}>Remove action</button
												>
											</div>
										{/each}
									{/if}
									<div>
										<button type="button" class="secondary" on:click={() => addAction(kbi)}
											>Add action</button
										>
									</div>
								</div>
							</li>
						{/each}
					</ul>
				{/if}
				<div class="row">
					<button
						type="button"
						class="secondary"
						on:click={startRecordingKey}
						disabled={recordingKey}>{recordingKey ? 'Press a key…' : 'Record key'}</button
					>
				</div>
			</div>
		</li>
	</ol>
	<h2>Player</h2>

	<div class="player">
		<div class="row">
			<button type="button" class="primary" on:click={playAll} disabled={playing}>Play</button>
			<button type="button" class="danger" on:click={stopPlayback} disabled={!playing}>Stop</button>
			<button type="button" class="secondary" on:click={toggleFullscreen}
				>{isFullscreen ? 'Exit Fullscreen' : 'Fullscreen'}</button
			>
			<div class="grow"></div>
			<div>
				{#if playing}
					{`Playing #${currentIndex + 1}`}
				{:else if previewingIndex != null}
					{`Previewing #${previewingIndex + 1}`}
				{:else}
					Idle
				{/if}
			</div>
		</div>
		<div class="player-stage" bind:this={stageEl}>
			{#if visualType === 'video'}
				<video bind:this={videoEl} src={visualUrl} muted={visualMuted} playsinline></video>
			{:else if visualType === 'image'}
				<img class="stage-image" bind:this={imgEl} src={visualUrl} alt="" />
			{/if}
			{#if audioUrl}
				<audio bind:this={audioEl} src={audioUrl}></audio>
			{/if}
		</div>
	</div>
</main>

<style>
	* {
		box-sizing: border-box;
	}
	:global(html),
	:global(body) {
		height: 100%;
	}
	:global(:root) {
		--bg: #f3f4f6;
		--text: #111827;
		--panel-bg: #ffffff;
		--panel-border: #e5e7eb;
		--muted-bg: #f3f4f6;
		--muted-border: #e5e7eb;
		--primary: #2563eb;
		--primary-contrast: #ffffff;
		--secondary-bg: #f3f4f6;
		--secondary-text: #111827;
		--secondary-border: #e5e7eb;
		--danger: #ef4444;
		--danger-hover: #dc2626;
		--stage-bg: #fafafa;
		--pill-bg: #f3f4f6;
		--pill-text: #374151;
		--pill-video-bg: #dbeafe;
		--pill-video-text: #1e40af;
		--pill-audio-bg: #e0e7ff;
		--pill-audio-text: #3730a3;
		--pill-image-bg: #fee2e2;
		--pill-image-text: #991b1b;
		--pill-script-bg: #ecfeff;
		--pill-script-text: #155e75;
		--pill-control-bg: #fef3c7;
		--pill-control-text: #92400e;
	}
	:global(html.dark) {
		/* Dark grey base inspired by provided palette */
		--bg: #303030; /* content background */
		--text: #ffffff; /* main text */
		--panel-bg: #424242; /* container background */
		--panel-border: rgba(255, 255, 255, 0.12); /* subtle divider */
		--muted-bg: #3a3a3a; /* muted surfaces */
		--muted-border: rgba(255, 255, 255, 0.12);

		/* Yellow accent for primary actions */
		--primary: #fbc02d; /* amber */
		--primary-contrast: rgba(0, 0, 0, 0.87); /* readable on yellow */

		--secondary-bg: #424242;
		--secondary-text: #ffffff;
		--secondary-border: rgba(255, 255, 255, 0.12);

		--danger: #ef4444;
		--danger-hover: #dc2626;
		--stage-bg: #303030;

		/* Pills: greys for media/script, yellow for control */
		--pill-bg: #3a3a3a;
		--pill-text: rgba(255, 255, 255, 0.7);
		--pill-video-bg: #3d3d3d;
		--pill-video-text: #ffffff;
		--pill-audio-bg: #3d3d3d;
		--pill-audio-text: #ffffff;
		--pill-image-bg: #3d3d3d;
		--pill-image-text: #ffffff;
		--pill-script-bg: #3d3d3d;
		--pill-script-text: #ffffff;
		--pill-control-bg: #fbc02d;
		--pill-control-text: rgba(0, 0, 0, 0.87);
	}
	:global(body) {
		width: 80%;
		margin: 0 auto;
		font-family:
			system-ui,
			-apple-system,
			Segoe UI,
			Roboto,
			Arial,
			Noto Sans,
			Ubuntu,
			Cantarell,
			Helvetica Neue,
			sans-serif;
		color: var(--text);
		background: var(--bg);
	}

	/* item layout styling is handled inside item components */

	.arrange-builder {
		margin-top: 0.5rem;
	}
	.arrange-add {
		align-items: center;
	}
	.row {
		flex-wrap: wrap;
		display: flex;
		gap: 0.5rem;
		margin: 0.5rem 0;
	}
	.row .grow {
		flex: 1 1 auto;
	}
	.ellipsis {
		white-space: nowrap;
		overflow: hidden;
		text-overflow: ellipsis;
	}
	.primary {
		background: var(--primary);
		color: var(--primary-contrast);
		border: none;
		padding: 0.7rem 1rem;
		border-radius: 0.5rem;
		cursor: pointer;
	}
	.secondary {
		background: var(--secondary-bg);
		color: var(--secondary-text);
		border: none;
		padding: 0.7rem 1rem;
		border-radius: 0.5rem;
		cursor: pointer;
	}
	.danger {
		background: var(--danger);
		color: #fff;
		border: none;
		padding: 0.5rem 1rem;
		border-radius: 0.5rem;
		cursor: pointer;
	}
	.danger:hover {
		background: var(--danger-hover);
	}

	.arrangement-list {
		list-style: none;
		padding: 0;
		margin: 0.5rem 0 0;
		display: flex;
		flex-direction: column;
		gap: 0.5rem;
	}
	.segment {
		border: 0.1rem solid var(--panel-border);
		border-radius: 0.5rem;
		background: var(--panel-bg);
	}
	.seg-head {
		background-color: var(--secondary-bg);
		display: flex;
		justify-content: space-between;
		align-items: center;
		padding: 0.5rem 1rem;
		border-bottom: 0.1rem solid var(--muted-border);
	}
	.seg-actions {
		display: flex;
		flex-wrap: wrap;
		gap: 0.5rem;
		justify-content: flex-end;
	}
	.seg-actions button {
		margin-left: 0.5rem;
	}
	.seg-body {
		padding: 1rem;
		display: flex;
		flex-direction: column;
		align-items: flex-start;
	}
	.pill {
		display: inline-block;
		font-size: 0.75rem;
		text-transform: uppercase;
		letter-spacing: 0.03em;
		padding: 0.1rem 0.5rem;
		border-radius: 9999px;
		background: var(--pill-bg);
		color: var(--pill-text);
		margin-bottom: 0.5rem;
	}
	.pill.video {
		background: var(--pill-video-bg);
		color: var(--pill-video-text);
	}
	.pill.audio {
		background: var(--pill-audio-bg);
		color: var(--pill-audio-text);
	}
	.pill.image {
		background: var(--pill-image-bg);
		color: var(--pill-image-text);
	}
	.pill.script {
		background: var(--pill-script-bg);
		color: var(--pill-script-text);
	}
	.pill.control {
		background: var(--pill-control-bg);
		color: var(--pill-control-text);
	}
	.pair-row {
		border-top: 0.1rem solid var(--muted-border);
		padding-top: 0.5rem;
		align-items: center;
	}
	.paired {
		flex: 1 1 auto;
	}

	/* keybinds */
	.keybinds {
		margin-top: 0.5rem;
	}
	.kb-list {
		list-style: none;
		padding: 0;
		margin: 0.5rem 0 0;
		display: flex;
		flex-direction: column;
		gap: 0.5rem;
	}
	.kb-item {
		border: 0.1rem solid var(--panel-border);
		border-radius: 0.5rem;
		background: var(--panel-bg);
	}
	.kb-head {
		display: flex;
		justify-content: space-between;
		align-items: center;
		background-color: var(--secondary-bg);
		padding: 0.5rem 1rem;
		border-bottom: 0.1rem solid var(--muted-border);
	}
	.kb-actions {
		flex-wrap: wrap;
		padding: 1rem;
		display: flex;
		flex-direction: column;
		gap: 0.5rem;
	}
	.kb-action {
		display: flex;
		gap: 0.5rem;
	}

	/* player */
	.player {
		margin-top: 0.5rem;
		padding: 1rem;
		border: 0.1rem solid var(--panel-border);
		border-radius: 0.5rem;
		background: var(--panel-bg);
	}
	.player-stage {
		margin-top: 0.5rem;
		border: 0.1rem dashed var(--panel-border);
		border-radius: 0.5rem;
		min-height: 12.5rem;
		display: flex;
		align-items: center;
		justify-content: center;
		background: var(--stage-bg);
		overflow: hidden;
	}
	.player-stage video {
		width: 100%;
		height: 100%;
		object-fit: contain;
	}
	.stage-image {
		width: 100%;
		height: 100%;
		object-fit: contain;
	}

	/* pseudo fullscreen */
	:global(html.pseudo-fs) body {
		overflow: hidden;
	}
	:global(html.pseudo-fs) .player-stage {
		position: fixed;
		inset: 0;
		width: 100vw;
		height: 100vh;
		margin: 0;
		border: none;
		border-radius: 0;
		z-index: 9999;
		background: var(--stage-bg);
	}
	:global(html.pseudo-fs) .player-stage video,
	:global(html.pseudo-fs) .player-stage .stage-image {
		width: 100%;
		height: 100%;
		object-fit: contain;
	}
	/* native fullscreen adjustments */
	:global(.player-stage:fullscreen) video,
	:global(.player-stage:fullscreen) .stage-image,
	:global(.player-stage:-webkit-full-screen) video,
	:global(.player-stage:-webkit-full-screen) .stage-image {
		width: 100%;
		height: 100%;
		object-fit: contain;
	}

	ul,
	ol {
		list-style: none;
		padding: 0;
		margin: 0;
		display: flex;
		gap: 1rem;
		flex-direction: column;
	}

	.media-lists {
		display: flex;
		flex-direction: column;
		margin-top: 0.5rem;
		justify-content: space-between;
		overflow: auto;
		max-height: 100vh;

		& > div {
			flex: 1 1 200px;
		}
	}

	.video-list,
	.audio-list,
	.image-list,
	.script-list {
		flex-direction: row;
		flex-wrap: wrap;
	}

	/* lists inside main should have no bullets */
	.app-main ul {
		list-style: none;
		padding-left: 0;
		margin-left: 0;
	}

	label {
		gap: 1rem;
		display: flex;
		flex-direction: column;
		align-items: flex-start;
	}

	input,
	button,
	select {
		transition: filter 0.1s ease-out;
	}
	/* form controls */
	input:hover,
	button:hover,
	select:hover {
		filter: brightness(1.3);
	}
	button:not(.primary):not(.secondary):not(.danger) {
		background: var(--secondary-bg);
		color: var(--secondary-text);
		border: 0.1rem solid var(--secondary-border);
		padding: 0.5rem 1rem;
		border-radius: 0.5rem;
		cursor: pointer;
	}
	input:not([type='checkbox']):not([type='radio']):not([type='range']),
	select {
		font: inherit;
		color: var(--text);
		background: var(--panel-bg);
		border: 0.1rem solid var(--panel-border);
		border-radius: 0.5rem;
		padding: 0.5rem 1rem;
	}
	input::placeholder {
		color: inherit;
		opacity: 0.7;
	}
	input:focus,
	select:focus {
		outline: 0.1rem solid var(--primary);
		outline-offset: 0.1rem;
		border-color: var(--primary);
	}
	input[disabled],
	select[disabled],
	button[disabled] {
		opacity: 0.6;
		cursor: not-allowed;
	}
	input[type='checkbox'],
	input[type='radio'] {
		accent-color: var(--primary);
	}
	/* make controls inside a flex grow label fill width */
	.row .grow input,
	.row .grow select {
		width: 100%;
	}
</style>
