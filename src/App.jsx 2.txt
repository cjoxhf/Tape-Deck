import { useState, useRef, useEffect, useMemo } from "react";
import {
  Plus, Play, Pause, SkipBack, SkipForward, Disc3, ListMusic,
  Palette, X, Trash2, Volume2, VolumeX, MoreVertical, Check,
  Music2, FolderPlus, ShieldCheck
} from "lucide-react";

const THEMES = {
  tape: {
    label: "Tape Deck", bg: "#1c1815", bgAlt: "#221d17", surface: "#28221a",
    surfaceHover: "#332b20", border: "#3c3327", text: "#f2e8d8", textDim: "#a8977c",
    accent: "#e8934a", accent2: "#c9612f", onAccent: "#1c1815", swatch: ["#1c1815", "#e8934a", "#c9612f"]
  },
  vinyl: {
    label: "Vinyl Noir", bg: "#0c0c0d", bgAlt: "#141415", surface: "#19191a",
    surfaceHover: "#232324", border: "#2b2b2c", text: "#f2ede4", textDim: "#8f887f",
    accent: "#c9433a", accent2: "#8a2620", onAccent: "#f2ede4", swatch: ["#0c0c0d", "#c9433a", "#8a2620"]
  },
  neon: {
    label: "Neon Wave", bg: "#100c22", bgAlt: "#171132", surface: "#1e1740",
    surfaceHover: "#28204f", border: "#332a5e", text: "#eef0ff", textDim: "#948dc4",
    accent: "#ff5fa8", accent2: "#5ff0ff", onAccent: "#100c22", swatch: ["#100c22", "#ff5fa8", "#5ff0ff"]
  },
  paper: {
    label: "Paper", bg: "#f4f1ea", bgAlt: "#ece7db", surface: "#ffffff",
    surfaceHover: "#f0ece0", border: "#ddd5c2", text: "#1c1a15", textDim: "#7a7460",
    accent: "#1c1a15", accent2: "#b3312c", onAccent: "#f4f1ea", swatch: ["#f4f1ea", "#1c1a15", "#b3312c"]
  }
};

function fmtTime(s) {
  if (!isFinite(s) || s < 0) return "0:00";
  const m = Math.floor(s / 60);
  const sec = Math.floor(s % 60).toString().padStart(2, "0");
  return `${m}:${sec}`;
}

function parseNameParts(filename) {
  const base = filename.replace(/\.[^/.]+$/, "");
  const cleaned = base.replace(/[_]+/g, " ").trim();
  const dashSplit = cleaned.split(/\s-\s/);
  if (dashSplit.length >= 2) {
    return { artist: dashSplit[0].trim(), title: dashSplit.slice(1).join(" - ").trim() };
  }
  return { artist: "Unknown artist", title: cleaned };
}

let idCounter = 1;
const nextId = () => idCounter++;

const LIB_KEY = "td-library-meta";
const PLAYLISTS_KEY = "td-playlists-meta";
const SESSION_KEY = "td-last-session";
const fileKey = (name, size) => `${name}::${size}`;

async function storageGet(key) {
  try {
    const raw = localStorage.getItem(key);
    return raw ? JSON.parse(raw) : null;
  } catch (e) {
    return null;
  }
}
async function storageSet(key, value) {
  try {
    localStorage.setItem(key, JSON.stringify(value));
  } catch (e) {
    // storage unavailable (e.g. private browsing) — app still works in-session
  }
}
async function storageDelete(key) {
  try {
    localStorage.removeItem(key);
  } catch (e) {}
}

export default function TapeDeck() {
  const [themeKey, setThemeKey] = useState("tape");
  const theme = THEMES[themeKey];
  const [themeMenuOpen, setThemeMenuOpen] = useState(false);

  const [tracks, setTracks] = useState([]);
  const [playlists, setPlaylists] = useState([]);
  const [view, setView] = useState({ type: "library" }); // {type:'library'} | {type:'playlist', id}
  const [rowMenuOpenId, setRowMenuOpenId] = useState(null);

  const [currentTrackId, setCurrentTrackId] = useState(null);
  const [isPlaying, setIsPlaying] = useState(false);
  const [progress, setProgress] = useState(0);
  const [duration, setDuration] = useState(0);
  const [volume, setVolume] = useState(0.8);
  const [muted, setMuted] = useState(false);

  const [newPlaylistName, setNewPlaylistName] = useState("");
  const [showNewPlaylistInput, setShowNewPlaylistInput] = useState(false);

  const [restoreState, setRestoreState] = useState(null); // { libraryMeta, playlistsMeta, session, matchedKeys: Set }
  const [loadedFromStorage, setLoadedFromStorage] = useState(false);

  const audioRef = useRef(null);
  const fileInputRef = useRef(null);
  const hydratedRef = useRef(false); // true once initial storage load completes, so we don't overwrite saved data with empty state
  const lastSaveRef = useRef(0);
  const sessionResumedRef = useRef(false);

  const queue = useMemo(() => {
    if (view.type === "library") return tracks;
    const pl = playlists.find((p) => p.id === view.id);
    if (!pl) return [];
    return pl.trackIds.map((tid) => tracks.find((t) => t.id === tid)).filter(Boolean);
  }, [view, tracks, playlists]);

  const currentTrack = tracks.find((t) => t.id === currentTrackId) || null;
  const currentIndexInQueue = queue.findIndex((t) => t.id === currentTrackId);

  // Load whatever we remember from last time, once, on mount.
  useEffect(() => {
    (async () => {
      const [libraryMeta, playlistsMeta, session] = await Promise.all([
        storageGet(LIB_KEY),
        storageGet(PLAYLISTS_KEY),
        storageGet(SESSION_KEY)
      ]);
      if (libraryMeta && libraryMeta.length) {
        setRestoreState({ libraryMeta, playlistsMeta: playlistsMeta || [], session: session || null });
      }
      hydratedRef.current = true;
      setLoadedFromStorage(true);
    })();
  }, []);

  // Keep the saved library + playlist structure in sync (metadata only — never the audio itself).
  useEffect(() => {
    if (!hydratedRef.current) return;
    const libMeta = tracks.map((t) => ({
      name: t.name, artist: t.artist, fileName: t.file.name, fileSize: t.file.size, duration: t.duration
    }));
    storageSet(LIB_KEY, libMeta);
  }, [tracks]);

  useEffect(() => {
    if (!hydratedRef.current) return;
    const plMeta = playlists.map((p) => ({
      name: p.name,
      trackRefs: p.trackIds
        .map((id) => tracks.find((t) => t.id === id))
        .filter(Boolean)
        .map((t) => ({ fileName: t.file.name, fileSize: t.file.size }))
    }));
    storageSet(PLAYLISTS_KEY, plMeta);
  }, [playlists, tracks]);

  useEffect(() => {
    const audio = audioRef.current;
    if (!audio) return;
    audio.volume = muted ? 0 : volume;
  }, [volume, muted]);

  useEffect(() => {
    const audio = audioRef.current;
    if (!audio || !currentTrack) return;
    if (audio.src !== currentTrack.url) {
      audio.src = currentTrack.url;
    }
    if (isPlaying) {
      audio.play().catch(() => setIsPlaying(false));
    }
  }, [currentTrackId]);

  useEffect(() => {
    const audio = audioRef.current;
    if (!audio) return;
    if (isPlaying) audio.play().catch(() => setIsPlaying(false));
    else audio.pause();
  }, [isPlaying]);

  function handleFiles(fileList) {
    const files = Array.from(fileList).filter((f) => f.type.startsWith("audio/") || /\.(mp3|wav|ogg|m4a|flac|aac)$/i.test(f.name));
    if (files.length === 0) return;

    const remembered = restoreState?.libraryMeta || [];
    const keyToNewTrack = {};

    const newTracks = files.map((file) => {
      const url = URL.createObjectURL(file);
      const match = remembered.find((m) => m.fileName === file.name && m.fileSize === file.size);
      const artist = match ? match.artist : parseNameParts(file.name).artist;
      const name = match ? match.name : parseNameParts(file.name).title;
      const track = { id: nextId(), file, url, name, artist, duration: match?.duration || null, addedAt: Date.now() };
      keyToNewTrack[fileKey(file.name, file.size)] = track.id;
      return track;
    });

    setTracks((prev) => [...prev, ...newTracks]);

    // Probe real duration in the background regardless of remembered value.
    newTracks.forEach((t) => {
      const probe = new Audio();
      probe.preload = "metadata";
      probe.src = t.url;
      probe.addEventListener("loadedmetadata", () => {
        setTracks((prev) => prev.map((p) => (p.id === t.id ? { ...p, duration: probe.duration } : p)));
      });
    });

    // Reconnect remembered playlists that reference these files.
    if (restoreState?.playlistsMeta?.length) {
      setPlaylists((prevPlaylists) => {
        const byName = new Map(prevPlaylists.map((p) => [p.name, p]));
        restoreState.playlistsMeta.forEach((pm) => {
          const matchedIds = pm.trackRefs
            .map((ref) => keyToNewTrack[fileKey(ref.fileName, ref.fileSize)])
            .filter(Boolean);
          if (matchedIds.length === 0) return;
          const existing = byName.get(pm.name);
          if (existing) {
            existing.trackIds = Array.from(new Set([...existing.trackIds, ...matchedIds]));
          } else {
            const pl = { id: nextId(), name: pm.name, trackIds: matchedIds };
            byName.set(pm.name, pl);
          }
        });
        return Array.from(byName.values());
      });
    }

    // Resume the exact song + timestamp we last remembered, if it just got re-added.
    const session = restoreState?.session;
    if (!sessionResumedRef.current && session) {
      const resumeId = keyToNewTrack[fileKey(session.fileName, session.fileSize)];
      if (resumeId) {
        sessionResumedRef.current = true;
        setCurrentTrackId(resumeId);
        setIsPlaying(false);
        const seekOnce = () => {
          if (audioRef.current) {
            audioRef.current.currentTime = session.position;
            setProgress(session.position);
          }
        };
        // audio.src updates asynchronously via the currentTrackId effect; give it a tick.
        setTimeout(seekOnce, 150);
      }
    }

    // Drop anything we've now successfully reconnected out of the "still missing" banner.
    setRestoreState((prev) => {
      if (!prev) return prev;
      const stillMissingLib = prev.libraryMeta.filter((m) => !keyToNewTrack[fileKey(m.fileName, m.fileSize)]);
      if (stillMissingLib.length === 0) return null;
      return { ...prev, libraryMeta: stillMissingLib };
    });

    if (!currentTrackId && !sessionResumedRef.current && newTracks.length) {
      setCurrentTrackId(newTracks[0].id);
    }
  }

  function forgetSavedData() {
    storageDelete(LIB_KEY);
    storageDelete(PLAYLISTS_KEY);
    storageDelete(SESSION_KEY);
    setRestoreState(null);
  }

  function playTrack(id) {
    if (id === currentTrackId) {
      setIsPlaying((p) => !p);
    } else {
      setCurrentTrackId(id);
      setIsPlaying(true);
      setProgress(0);
    }
  }

  function playAdjacent(dir) {
    if (queue.length === 0) return;
    let idx = currentIndexInQueue;
    if (idx === -1) idx = 0;
    else idx = (idx + dir + queue.length) % queue.length;
    setCurrentTrackId(queue[idx].id);
    setIsPlaying(true);
    setProgress(0);
  }

  function onTimeUpdate() {
    const audio = audioRef.current;
    if (!audio) return;
    setProgress(audio.currentTime);
    if (audio.duration) setDuration(audio.duration);
    const now = Date.now();
    if (currentTrack && now - lastSaveRef.current > 4000) {
      lastSaveRef.current = now;
      storageSet(SESSION_KEY, {
        fileName: currentTrack.file.name, fileSize: currentTrack.file.size,
        name: currentTrack.name, artist: currentTrack.artist, position: audio.currentTime
      });
    }
  }

  function saveSessionNow() {
    const audio = audioRef.current;
    if (!audio || !currentTrack) return;
    storageSet(SESSION_KEY, {
      fileName: currentTrack.file.name, fileSize: currentTrack.file.size,
      name: currentTrack.name, artist: currentTrack.artist, position: audio.currentTime
    });
  }

  useEffect(() => {
    if (!isPlaying) saveSessionNow();
  }, [isPlaying]);

  useEffect(() => {
    const handler = () => saveSessionNow();
    window.addEventListener("beforeunload", handler);
    document.addEventListener("visibilitychange", handler);
    return () => {
      window.removeEventListener("beforeunload", handler);
      document.removeEventListener("visibilitychange", handler);
    };
  }, [currentTrackId]);

  function onSeek(e) {
    const audio = audioRef.current;
    if (!audio || !duration) return;
    const val = Number(e.target.value);
    audio.currentTime = val;
    setProgress(val);
  }

  function onEnded() {
    playAdjacent(1);
  }

  function removeTrack(id) {
    setTracks((prev) => prev.filter((t) => t.id !== id));
    setPlaylists((prev) => prev.map((p) => ({ ...p, trackIds: p.trackIds.filter((tid) => tid !== id) })));
    if (currentTrackId === id) {
      setIsPlaying(false);
      setCurrentTrackId(null);
    }
  }

  function createPlaylist() {
    const name = newPlaylistName.trim();
    if (!name) return;
    const pl = { id: nextId(), name, trackIds: [] };
    setPlaylists((prev) => [...prev, pl]);
    setNewPlaylistName("");
    setShowNewPlaylistInput(false);
    setView({ type: "playlist", id: pl.id });
  }

  function toggleTrackInPlaylist(playlistId, trackId) {
    setPlaylists((prev) =>
      prev.map((p) => {
        if (p.id !== playlistId) return p;
        const has = p.trackIds.includes(trackId);
        return { ...p, trackIds: has ? p.trackIds.filter((id) => id !== trackId) : [...p.trackIds, trackId] };
      })
    );
  }

  function deletePlaylist(id) {
    setPlaylists((prev) => prev.filter((p) => p.id !== id));
    if (view.type === "playlist" && view.id === id) setView({ type: "library" });
  }

  const activeTitle = view.type === "library" ? "Library" : playlists.find((p) => p.id === view.id)?.name || "Playlist";

  return (
    <div
      className="w-full h-screen flex flex-col overflow-hidden"
      style={{ background: theme.bg, color: theme.text, fontFamily: "'Inter', system-ui, sans-serif" }}
    >
      <style>{`
        @import url('https://fonts.googleapis.com/css2?family=Fraunces:opsz,wght@9..144,500;9..144,700&family=JetBrains+Mono:wght@400;500;600&family=Inter:wght@400;500;600&display=swap');
        .td-mono { font-family: 'JetBrains Mono', monospace; }
        .td-display { font-family: 'Fraunces', serif; }
        .td-scrollbar::-webkit-scrollbar { width: 8px; height: 8px; }
        .td-scrollbar::-webkit-scrollbar-thumb { background: ${theme.border}; border-radius: 8px; }
        .td-reel { animation: td-spin 3s linear infinite; }
        @media (prefers-reduced-motion: reduce) { .td-reel { animation: none; } }
        @keyframes td-spin { from { transform: rotate(0deg); } to { transform: rotate(360deg); } }
        .td-row:hover { background: ${theme.surfaceHover}; }
        input[type=range].td-range { -webkit-appearance: none; height: 4px; border-radius: 2px; background: ${theme.border}; }
        input[type=range].td-range::-webkit-slider-thumb { -webkit-appearance: none; width: 12px; height: 12px; border-radius: 50%; background: ${theme.accent}; cursor: pointer; margin-top: -4px; }
        input[type=range].td-range::-moz-range-thumb { width: 12px; height: 12px; border-radius: 50%; background: ${theme.accent}; border: none; cursor: pointer; }
      `}</style>

      {/* Header */}
      <div
        className="flex items-center justify-between px-5 py-4 shrink-0"
        style={{ borderBottom: `1px solid ${theme.border}` }}
      >
        <div className="flex items-center gap-2.5">
          <Disc3 size={22} style={{ color: theme.accent }} className={isPlaying ? "td-reel" : ""} />
          <span className="td-display text-lg tracking-tight" style={{ fontWeight: 700 }}>Tape Deck</span>
        </div>
        <div className="flex items-center gap-2">
          <input
            ref={fileInputRef}
            type="file"
            accept="audio/*"
            multiple
            className="hidden"
            onChange={(e) => { handleFiles(e.target.files); e.target.value = ""; }}
          />
          <button
            onClick={() => fileInputRef.current?.click()}
            className="flex items-center gap-1.5 px-3.5 py-2 rounded-full text-sm font-medium transition-transform active:scale-95"
            style={{ background: theme.accent, color: theme.onAccent }}
          >
            <Plus size={16} strokeWidth={2.5} /> Upload
          </button>
          <div className="relative">
            <button
              onClick={() => setThemeMenuOpen((v) => !v)}
              aria-label="Change theme"
              className="p-2 rounded-full transition-colors"
              style={{ background: theme.surface, border: `1px solid ${theme.border}` }}
            >
              <Palette size={16} style={{ color: theme.textDim }} />
            </button>
            {themeMenuOpen && (
              <div
                className="absolute right-0 mt-2 w-52 rounded-lg p-1.5 z-20 td-scrollbar"
                style={{ background: theme.surface, border: `1px solid ${theme.border}` }}
              >
                {Object.entries(THEMES).map(([key, t]) => (
                  <button
                    key={key}
                    onClick={() => { setThemeKey(key); setThemeMenuOpen(false); }}
                    className="w-full flex items-center gap-2.5 px-2.5 py-2 rounded-md text-sm td-row"
                  >
                    <span className="flex shrink-0 rounded-full overflow-hidden" style={{ width: 16, height: 16, border: `1px solid ${theme.border}` }}>
                      {t.swatch.map((c, i) => <span key={i} style={{ background: c, width: "33.33%" }} />)}
                    </span>
                    <span style={{ color: theme.text }}>{t.label}</span>
                    {themeKey === key && <Check size={14} className="ml-auto" style={{ color: theme.accent }} />}
                  </button>
                ))}
              </div>
            )}
          </div>
        </div>
      </div>

      {/* Body */}
      <div className="flex flex-1 min-h-0">
        {/* Sidebar */}
        <div
          className="w-56 shrink-0 flex flex-col p-3 gap-1 td-scrollbar overflow-y-auto"
          style={{ borderRight: `1px solid ${theme.border}`, background: theme.bgAlt }}
        >
          <button
            onClick={() => setView({ type: "library" })}
            className="flex items-center gap-2.5 px-3 py-2 rounded-md text-sm text-left"
            style={{
              background: view.type === "library" ? theme.surface : "transparent",
              color: view.type === "library" ? theme.text : theme.textDim,
              fontWeight: view.type === "library" ? 600 : 500
            }}
          >
            <Music2 size={16} /> Library
            <span className="td-mono ml-auto text-xs" style={{ color: theme.textDim }}>{tracks.length}</span>
          </button>

          <div className="flex items-center justify-between px-3 pt-4 pb-1">
            <span className="text-xs uppercase tracking-wider" style={{ color: theme.textDim }}>Playlists</span>
            <button onClick={() => setShowNewPlaylistInput((v) => !v)} aria-label="New playlist">
              <FolderPlus size={15} style={{ color: theme.textDim }} />
            </button>
          </div>

          {showNewPlaylistInput && (
            <div className="px-1 pb-1 flex gap-1">
              <input
                autoFocus
                value={newPlaylistName}
                onChange={(e) => setNewPlaylistName(e.target.value)}
                onKeyDown={(e) => e.key === "Enter" && createPlaylist()}
                placeholder="Playlist name"
                className="flex-1 min-w-0 text-sm px-2 py-1.5 rounded-md outline-none"
                style={{ background: theme.surface, border: `1px solid ${theme.border}`, color: theme.text }}
              />
              <button onClick={createPlaylist} className="px-2 rounded-md text-xs font-medium" style={{ background: theme.accent, color: theme.onAccent }}>Add</button>
            </div>
          )}

          {playlists.length === 0 && !showNewPlaylistInput && (
            <p className="px-3 text-xs leading-relaxed" style={{ color: theme.textDim }}>No playlists yet. Tap + to start one.</p>
          )}

          {playlists.map((p) => (
            <div key={p.id} className="group relative flex items-center">
              <button
                onClick={() => setView({ type: "playlist", id: p.id })}
                className="flex-1 min-w-0 flex items-center gap-2.5 px-3 py-2 rounded-md text-sm text-left"
                style={{
                  background: view.type === "playlist" && view.id === p.id ? theme.surface : "transparent",
                  color: view.type === "playlist" && view.id === p.id ? theme.text : theme.textDim,
                  fontWeight: view.type === "playlist" && view.id === p.id ? 600 : 500
                }}
              >
                <ListMusic size={16} className="shrink-0" />
                <span className="truncate">{p.name}</span>
                <span className="td-mono ml-auto text-xs shrink-0">{p.trackIds.length}</span>
              </button>
              <button
                onClick={() => deletePlaylist(p.id)}
                aria-label="Delete playlist"
                className="hidden group-hover:block absolute right-1 p-1 rounded"
                style={{ color: theme.textDim, background: theme.bgAlt }}
              >
                <X size={12} />
              </button>
            </div>
          ))}

          <div className="mt-auto pt-4 flex flex-col gap-2 px-1">
            <div className="flex items-start gap-1.5">
              <ShieldCheck size={13} className="shrink-0 mt-0.5" style={{ color: theme.textDim }} />
              <p className="text-[11px] leading-snug" style={{ color: theme.textDim }}>
                Songs never leave this tab. Only titles, playlist names, and your last timestamp are remembered — privately, in this browser, never shared.
              </p>
            </div>
            {(tracks.length > 0 || playlists.length > 0) && (
              <button
                onClick={forgetSavedData}
                className="self-start text-[11px] underline"
                style={{ color: theme.textDim }}
              >
                Forget saved data
              </button>
            )}
          </div>
        </div>

        {/* Track list */}
        <div className="flex-1 min-w-0 flex flex-col">
          <div className="px-6 pt-5 pb-2 flex items-center justify-between shrink-0">
            <h2 className="td-display text-xl" style={{ fontWeight: 700 }}>{activeTitle}</h2>
            {view.type === "playlist" && (
              <span className="text-xs" style={{ color: theme.textDim }}>Use the ⋮ menu on a library track to add it here</span>
            )}
          </div>

          {restoreState?.libraryMeta?.length > 0 && (
            <div
              className="mx-3 mb-3 rounded-lg px-4 py-3 flex items-start gap-3"
              style={{ background: theme.surface, border: `1px solid ${theme.border}` }}
            >
              <Music2 size={16} className="mt-0.5 shrink-0" style={{ color: theme.accent }} />
              <div className="flex-1 min-w-0">
                <p className="text-sm" style={{ fontWeight: 600 }}>Picking up where you left off</p>
                <p className="text-xs mt-0.5 leading-relaxed" style={{ color: theme.textDim }}>
                  {restoreState.session ? (
                    <>Last playing <span style={{ color: theme.text }}>{restoreState.session.name}</span> at {fmtTime(restoreState.session.position)}. </>
                  ) : null}
                  Re-upload the same {restoreState.libraryMeta.length} file{restoreState.libraryMeta.length === 1 ? "" : "s"} and they'll reconnect to their playlists automatically{restoreState.session ? ", and playback will resume at the exact spot" : ""}.
                </p>
              </div>
              <button onClick={forgetSavedData} className="text-xs shrink-0" style={{ color: theme.textDim }}>Dismiss</button>
            </div>
          )}

          <div className="flex-1 overflow-y-auto px-3 pb-4 td-scrollbar">
            {queue.length === 0 ? (
              <div className="h-full flex flex-col items-center justify-center gap-4 py-16 text-center px-6">
                <div
                  className="rounded-full p-5"
                  style={{ background: theme.surface, border: `1px dashed ${theme.border}` }}
                >
                  <Music2 size={28} style={{ color: theme.textDim }} />
                </div>
                <div>
                  <p className="font-medium" style={{ color: theme.text }}>
                    {view.type === "library" ? "Nothing uploaded yet" : "This playlist is empty"}
                  </p>
                  <p className="text-sm mt-1" style={{ color: theme.textDim }}>
                    {view.type === "library" ? "Upload songs from your device to get started." : "Add tracks from your Library using the ⋮ menu."}
                  </p>
                </div>
                {view.type === "library" && (
                  <button
                    onClick={() => fileInputRef.current?.click()}
                    className="flex items-center gap-2 px-4 py-2.5 rounded-full text-sm font-medium mt-1"
                    style={{ background: theme.accent, color: theme.onAccent }}
                  >
                    <Plus size={16} strokeWidth={2.5} /> Upload
                  </button>
                )}
              </div>
            ) : (
              <table className="w-full text-sm border-collapse">
                <tbody>
                  {queue.map((t, i) => {
                    const active = t.id === currentTrackId;
                    return (
                      <tr
                        key={t.id}
                        onDoubleClick={() => playTrack(t.id)}
                        className="td-row group cursor-pointer"
                        style={{ color: active ? theme.accent : theme.text }}
                      >
                        <td className="w-10 pl-3 py-2.5 td-mono text-xs" style={{ color: theme.textDim }}>
                          <button onClick={() => playTrack(t.id)} className="w-6 flex items-center justify-center" aria-label="Play">
                            {active && isPlaying ? (
                              <Pause size={13} style={{ color: theme.accent }} />
                            ) : active ? (
                              <Play size={13} style={{ color: theme.accent }} />
                            ) : (
                              <span className="group-hover:hidden">{String(i + 1).padStart(3, "0")}</span>
                            )}
                            {!active && <Play size={13} className="hidden group-hover:block" />}
                          </button>
                        </td>
                        <td className="py-2.5 pr-3">
                          <p className="truncate max-w-[380px]" style={{ fontWeight: active ? 600 : 500 }}>{t.name}</p>
                          <p className="truncate max-w-[380px] text-xs" style={{ color: theme.textDim }}>{t.artist}</p>
                        </td>
                        <td className="td-mono text-xs pr-3 whitespace-nowrap" style={{ color: theme.textDim }}>
                          {t.duration ? fmtTime(t.duration) : "--:--"}
                        </td>
                        <td className="w-10 pr-3 relative">
                          <button
                            onClick={() => setRowMenuOpenId(rowMenuOpenId === t.id ? null : t.id)}
                            className="p-1.5 rounded opacity-0 group-hover:opacity-100"
                            style={{ color: theme.textDim }}
                            aria-label="More options"
                          >
                            <MoreVertical size={15} />
                          </button>
                          {rowMenuOpenId === t.id && (
                            <div
                              className="absolute right-2 top-8 w-52 rounded-lg p-1.5 z-20"
                              style={{ background: theme.surface, border: `1px solid ${theme.border}` }}
                              onMouseLeave={() => setRowMenuOpenId(null)}
                            >
                              <p className="px-2.5 pt-1 pb-1.5 text-[11px] uppercase tracking-wider" style={{ color: theme.textDim }}>Add to playlist</p>
                              {playlists.length === 0 && (
                                <p className="px-2.5 pb-2 text-xs" style={{ color: theme.textDim }}>No playlists yet</p>
                              )}
                              {playlists.map((p) => (
                                <button
                                  key={p.id}
                                  onClick={() => toggleTrackInPlaylist(p.id, t.id)}
                                  className="w-full flex items-center gap-2 px-2.5 py-1.5 rounded-md text-sm td-row text-left"
                                  style={{ color: theme.text }}
                                >
                                  <span className="truncate flex-1">{p.name}</span>
                                  {p.trackIds.includes(t.id) && <Check size={13} style={{ color: theme.accent }} />}
                                </button>
                              ))}
                              <div style={{ borderTop: `1px solid ${theme.border}` }} className="mt-1 pt-1">
                                <button
                                  onClick={() => { removeTrack(t.id); setRowMenuOpenId(null); }}
                                  className="w-full flex items-center gap-2 px-2.5 py-1.5 rounded-md text-sm td-row text-left"
                                  style={{ color: theme.accent2 }}
                                >
                                  <Trash2 size={13} /> Remove from library
                                </button>
                              </div>
                            </div>
                          )}
                        </td>
                      </tr>
                    );
                  })}
                </tbody>
              </table>
            )}
          </div>
        </div>
      </div>

      {/* Player bar */}
      <div
        className="shrink-0 px-5 py-3 flex items-center gap-4"
        style={{ borderTop: `1px solid ${theme.border}`, background: theme.bgAlt }}
      >
        <audio
          ref={audioRef}
          onTimeUpdate={onTimeUpdate}
          onLoadedMetadata={onTimeUpdate}
          onEnded={onEnded}
        />
        <div className="w-44 shrink-0 min-w-0">
          {currentTrack ? (
            <>
              <p className="text-sm truncate" style={{ fontWeight: 600 }}>{currentTrack.name}</p>
              <p className="text-xs truncate" style={{ color: theme.textDim }}>{currentTrack.artist}</p>
            </>
          ) : (
            <p className="text-xs" style={{ color: theme.textDim }}>Nothing playing</p>
          )}
        </div>

        <div className="flex-1 flex items-center gap-3 min-w-0">
          <button onClick={() => playAdjacent(-1)} disabled={queue.length === 0} style={{ color: theme.text }}>
            <SkipBack size={17} fill={theme.text} />
          </button>
          <button
            onClick={() => currentTrack ? setIsPlaying((p) => !p) : queue[0] && playTrack(queue[0].id)}
            disabled={queue.length === 0}
            className="rounded-full p-2.5"
            style={{ background: theme.accent, color: theme.onAccent }}
          >
            {isPlaying ? <Pause size={16} fill={theme.onAccent} /> : <Play size={16} fill={theme.onAccent} />}
          </button>
          <button onClick={() => playAdjacent(1)} disabled={queue.length === 0} style={{ color: theme.text }}>
            <SkipForward size={17} fill={theme.text} />
          </button>

          <span className="td-mono text-xs w-9 text-right shrink-0" style={{ color: theme.textDim }}>{fmtTime(progress)}</span>
          <input
            type="range"
            className="td-range flex-1 min-w-0"
            min={0}
            max={duration || 0}
            value={Math.min(progress, duration || 0)}
            onChange={onSeek}
            disabled={!currentTrack}
          />
          <span className="td-mono text-xs w-9 shrink-0" style={{ color: theme.textDim }}>{fmtTime(duration)}</span>
        </div>

        <div className="flex items-center gap-2 w-28 shrink-0 justify-end">
          <button onClick={() => setMuted((m) => !m)} style={{ color: theme.textDim }}>
            {muted || volume === 0 ? <VolumeX size={16} /> : <Volume2 size={16} />}
          </button>
          <input
            type="range"
            className="td-range w-16"
            min={0}
            max={1}
            step={0.01}
            value={muted ? 0 : volume}
            onChange={(e) => { setVolume(Number(e.target.value)); setMuted(false); }}
          />
        </div>
      </div>
    </div>
  );
}
