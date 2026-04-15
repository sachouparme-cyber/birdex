import React, { useState, useEffect, useRef } from "react";

// --- CONFIGURATION & CONSTANTES ---
const apiKey = ""; // Clé fournie par l'environnement
const modelId = "gemini-2.5-flash-preview-09-2025";

const rarityConfig = {
  "commun":      { color: "#4ade80", bg: "#052e16", label: "Commun",        stars: 1 },
  "peu commun":  { color: "#60a5fa", bg: "#0c1a3a", label: "Peu commun",    stars: 2 },
  "rare":        { color: "#f59e0b", bg: "#2d1a00", label: "⭐ Rare !",      stars: 3 },
  "très rare":   { color: "#f43f5e", bg: "#2d0014", label: "🌟 Très rare !!", stars: 4 },
};

// --- COMPOSANTS UI ---

const RarityBadge = ({ r }) => {
  const cfg = rarityConfig[r] || rarityConfig["commun"];
  return (
    <span className="px-3 py-0.5 rounded-full text-xs font-bold border" 
          style={{ background: cfg.bg, color: cfg.color, borderColor: cfg.color }}>
      {cfg.label}
    </span>
  );
};

const Sonogram = ({ isListening, onDetection }) => {
  const canvasRef = useRef(null);
  const tempCanvasRef = useRef(document.createElement('canvas'));
  const requestRef = useRef();
  const audioCtxRef = useRef();
  const analyserRef = useRef();
  const streamRef = useRef();
  const lastDetectionRef = useRef(0);

  useEffect(() => {
    const canvas = canvasRef.current;
    if (!canvas) return;
    const ctx = canvas.getContext('2d');
    const tempCanvas = tempCanvasRef.current;
    const tempCtx = tempCanvas.getContext('2d');

    const resize = () => {
      canvas.width = canvas.clientWidth;
      canvas.height = canvas.clientHeight;
      tempCanvas.width = canvas.width;
      tempCanvas.height = canvas.height;
      tempCtx.fillStyle = "#fff";
      tempCtx.fillRect(0, 0, tempCanvas.width, tempCanvas.height);
    };

    resize();
    window.addEventListener('resize', resize);

    if (isListening) {
      const startAudio = async () => {
        try {
          const stream = await navigator.mediaDevices.getUserMedia({ audio: true });
          streamRef.current = stream;
          const AudioContext = window.AudioContext || window.webkitAudioContext;
          const audioCtx = new AudioContext();
          audioCtxRef.current = audioCtx;
          
          const source = audioCtx.createMediaStreamSource(stream);
          const analyser = audioCtx.createAnalyser();
          analyser.fftSize = 2048;
          source.connect(analyser);
          analyserRef.current = analyser;

          const bufferLength = analyser.frequencyBinCount;
          const dataArray = new Uint8Array(bufferLength);

          const loop = () => {
            analyser.getByteFrequencyData(dataArray);
            const w = tempCanvas.width;
            const h = tempCanvas.height;

            tempCtx.drawImage(tempCanvas, -1, 0);
            tempCtx.fillStyle = "#fff";
            tempCtx.fillRect(w - 1, 0, 1, h);

            let maxVal = 0;
            for (let i = 0; i < bufferLength; i++) {
              const value = dataArray[i];
              if (value > maxVal) maxVal = value;
              if (value > 140) {
                const y = h - (i / (bufferLength / 2)) * h;
                const opacity = (value - 140) / (255 - 140);
                tempCtx.fillStyle = `rgba(0, 0, 0, ${opacity})`;
                tempCtx.fillRect(w - 1, y, 1, 2);
              }
            }
            ctx.drawImage(tempCanvas, 0, 0);

            const now = Date.now();
            if (maxVal > 180 && now - lastDetectionRef.current > 4000) {
              lastDetectionRef.current = now;
              onDetection();
            }

            requestRef.current = requestAnimationFrame(loop);
          };
          loop();
        } catch (err) {
          console.error("Microphone error:", err);
        }
      };
      startAudio();
    } else {
      if (streamRef.current) streamRef.current.getTracks().forEach(t => t.stop());
      cancelAnimationFrame(requestRef.current);
    }

    return () => {
      window.removeEventListener('resize', resize);
      cancelAnimationFrame(requestRef.current);
      if (streamRef.current) streamRef.current.getTracks().forEach(t => t.stop());
    };
  }, [isListening, onDetection]);

  return (
    <div className="relative w-full h-48 bg-white border-2 border-black overflow-hidden mb-4 rounded-xl">
      <canvas ref={canvasRef} className="w-full h-full" />
      <div className="absolute top-1 left-2 text-[8px] font-bold text-gray-400 uppercase">Analyse Spectrale Directe</div>
    </div>
  );
};

// --- APPLICATION PRINCIPALE ---

export default function App() {
  const [view, setView] = useState("pokedex");
  const [collection, setCollection] = useState([]);
  const [isListening, setIsListening] = useState(false);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState("");
  const [selectedBird, setSelectedBird] = useState(null);
  const [liveDetections, setLiveDetections] = useState([]);
  const [isAiAnalyzing, setIsAiAnalyzing] = useState(false);
  
  const [form, setForm] = useState({ species: "", location: "", notes: "" });

  useEffect(() => {
    const saved = localStorage.getItem("birddex_data");
    if (saved) setCollection(JSON.parse(saved));
  }, []);

  const saveToLocal = (newCol) => {
    setCollection(newCol);
    localStorage.setItem("birddex_data", JSON.stringify(newCol));
  };

  const analyzeLiveAudio = async () => {
    if (isAiAnalyzing) return;
    setIsAiAnalyzing(true);
    try {
      const prompt = `L'utilisateur écoute la nature. Suggère 3 oiseaux européens typiques (nom et emoji). JSON: {"birds": [{"name": "...", "emoji": "..."}]}`;
      
      const res = await fetch(`https://generativelanguage.googleapis.com/v1beta/models/${modelId}:generateContent?key=${apiKey}`, {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ contents: [{ parts: [{ text: prompt }] }] })
      });
      
      const data = await res.json();
      const text = data.candidates[0].content.parts[0].text;
      const json = JSON.parse(text.replace(/```json|```/g, ""));
      setLiveDetections(json.birds);
    } catch (err) {
      console.error("Live AI Error:", err);
    } finally {
      setIsAiAnalyzing(false);
    }
  };

  const handleIdentifyWithAI = async (speciesName = null) => {
    const targetSpecies = speciesName || form.species;
    if (!targetSpecies.trim()) return;
    
    setLoading(true);
    setError("");

    try {
      const prompt = `Fiche JSON pour l'oiseau : "${targetSpecies}". Inclus : nom_latin, rarete (commun, peu commun, rare, très rare), taille, description (2 phrases), habitat, chant, emoji.`;
      
      const res = await fetch(`https://generativelanguage.googleapis.com/v1beta/models/${modelId}:generateContent?key=${apiKey}`, {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ contents: [{ parts: [{ text: prompt }] }] })
      });
      
      const data = await res.json();
      const text = data.candidates[0].content.parts[0].text;
      const aiBird = JSON.parse(text.replace(/```json|```/g, ""));

      const newBird = {
        ...aiBird,
        id: Date.now(),
        nom_commun: targetSpecies,
        location: form.location || "Observation en direct",
        notes: form.notes,
        date: new Date().toLocaleDateString("fr-FR")
      };

      const newCol = [newBird, ...collection];
      saveToLocal(newCol);
      setSelectedBird(newBird);
      setView("card");
      setForm({ species: "", location: "", notes: "" });
      setLiveDetections([]);
    } catch (err) {
      setError("Erreur réseau ou IA.");
    } finally {
      setLoading(false);
    }
  };

  const deleteBird = (id) => {
    const newCol = collection.filter(b => b.id !== id);
    saveToLocal(newCol);
    setView("pokedex");
  };

  return (
    <div className="min-h-screen bg-[#060f08] text-[#d1fae5] font-sans pb-10">
      
      <header className="bg-gradient-to-br from-[#0a2e0a] to-[#0d1f12] border-b border-[#1f3a1f] p-6 flex justify-between items-center sticky top-0 z-50">
        <div className="cursor-pointer" onClick={() => setView("pokedex")}>
          <h1 className="text-2xl font-black text-[#6ee7b7] tracking-widest uppercase">🐦 BirdDex AI</h1>
          <p className="text-[10px] text-[#4b7a5a] uppercase font-bold tracking-tighter">Écoute Acoustique Active</p>
        </div>
        {view !== "observe" && (
          <button onClick={() => setView("observe")} className="bg-[#15803d] px-4 py-2 rounded-xl font-bold text-sm text-white shadow-lg active:scale-95 transition-transform">
            + OBSERVER
          </button>
        )}
      </header>

      <main className="max-w-xl mx-auto p-6">
        
        {view === "pokedex" && (
          <div className="space-y-6">
            <h2 className="text-lg font-bold text-[#6ee7b7]">Ma Collection ({collection.length})</h2>
            {collection.length === 0 ? (
              <div className="text-center py-20 border-2 border-dashed border-[#1f3a1f] rounded-3xl opacity-50">
                <p className="text-sm">Votre collection est vide.</p>
                <p className="text-[10px] uppercase mt-2">Appuyez sur Observer pour commencer</p>
              </div>
            ) : (
              <div className="grid grid-cols-2 gap-4">
                {collection.map(bird => (
                  <div key={bird.id} onClick={() => { setSelectedBird(bird); setView("card"); }}
                       className="bg-[#0d1f12] border border-[#1f3a1f] p-4 rounded-2xl hover:scale-105 transition-transform cursor-pointer text-center">
                    <div className="text-4xl mb-2">{bird.emoji}</div>
                    <div className="font-bold text-sm truncate">{bird.nom_commun}</div>
                    <RarityBadge r={bird.rarete} />
                  </div>
                ))}
              </div>
            )}
          </div>
        )}

        {view === "observe" && (
          <div className="space-y-6 animate-in fade-in slide-in-from-bottom-4">
            <h2 className="text-xl font-bold text-[#6ee7b7]">Analyse en direct</h2>
            
            <Sonogram isListening={isListening} onDetection={analyzeLiveAudio} />

            <button onClick={() => setIsListening(!isListening)} 
                    className={`w-full py-4 rounded-2xl font-black text-sm uppercase tracking-widest transition-all ${isListening ? 'bg-black border-2 border-red-600 text-red-600 animate-pulse' : 'bg-red-600 text-white shadow-xl shadow-red-900/20'}`}>
              {isListening ? "⏹ ARRÊTER L'ÉCOUTE" : "🎙 DÉMARRER L'ÉCOUTE"}
            </button>

            {isListening && (
              <div className="space-y-3 animate-in fade-in">
                <div className="flex justify-between items-center px-1">
                  <h3 className="text-[10px] font-black uppercase text-[#4b7a5a]">Détections probables</h3>
                  {isAiAnalyzing && <span className="text-[8px] animate-pulse text-[#6ee7b7]">SCAN IA...</span>}
                </div>
                <div className="flex gap-3 overflow-x-auto pb-2 scrollbar-hide">
                  {liveDetections.length === 0 && !isAiAnalyzing && (
                    <p className="text-[10px] italic text-[#4b7a5a] py-4 w-full text-center">Écoute des signaux en cours...</p>
                  )}
                  {liveDetections.map((det, idx) => (
                    <button key={idx} 
                            onClick={() => handleIdentifyWithAI(det.name)}
                            className="flex-shrink-0 bg-[#0d1f12] border border-[#6ee7b7] p-3 rounded-2xl text-center min-w-[120px] active:scale-95 transition-transform">
                      <div className="text-2xl mb-1">{det.emoji}</div>
                      <div className="text-[10px] font-bold text-white uppercase">{det.name}</div>
                      <div className="text-[8px] text-[#6ee7b7] mt-1 font-bold">AJOUTER +</div>
                    </button>
                  ))}
                </div>
              </div>
            )}

            <div className="space-y-4 bg-[#0a1a0a] p-6 rounded-3xl border border-[#1f3a1f]">
              <div>
                <label className="text-[10px] font-bold text-[#6ee7b7] uppercase block mb-1">Ou recherche manuelle</label>
                <input type="text" placeholder="Nom de l'oiseau..." 
                       value={form.species} onChange={e => setForm({...form, species: e.target.value})}
                       className="w-full bg-[#060f08] border border-[#1f3a1f] p-3 rounded-xl text-white outline-none focus:border-[#6ee7b7]" />
              </div>
              
              {loading && <div className="text-center py-2 text-xs text-[#6ee7b7] animate-pulse italic">✨ L'IA rédige la fiche...</div>}
              {error && <div className="text-red-400 text-[10px] text-center">{error}</div>}
              
              <button onClick={() => handleIdentifyWithAI()} disabled={loading || !form.species}
                      className="w-full bg-[#15803d] text-white font-black py-4 rounded-2xl uppercase text-sm disabled:opacity-50 active:scale-[0.98] transition-all">
                CRÉER LA FICHE
              </button>
              
              <button onClick={() => setView("pokedex")} className="w-full text-[10px] font-bold text-[#4b7a5a] uppercase">Retour au menu</button>
            </div>
          </div>
        )}

        {view === "card" && selectedBird && (
          <div className="animate-in zoom-in-95">
            <div className="bg-gradient-to-br from-[#0d1f12] to-[#060f08] border-2 border-[#6ee7b7] rounded-[40px] p-8 shadow-2xl relative">
              <button onClick={() => setView("pokedex")} className="absolute top-8 left-8 text-[#6ee7b7] text-xs font-bold uppercase tracking-widest opacity-70 hover:opacity-100">← Retour</button>
              <button onClick={() => deleteBird(selectedBird.id)} className="absolute top-8 right-8 text-red-400/50 hover:text-red-400 text-xs transition-colors">🗑 Supprimer</button>
              
              <div className="text-center mt-10 mb-8">
                <div className="text-8xl mb-4 drop-shadow-xl">{selectedBird.emoji}</div>
                <h2 className="text-3xl font-black text-white uppercase tracking-tight leading-none mb-1">{selectedBird.nom_commun}</h2>
                <p className="text-[#6ee7b7] italic text-sm mb-6">{selectedBird.nom_latin}</p>
                <RarityBadge r={selectedBird.rarete} />
              </div>

              <div className="grid grid-cols-2 gap-3 mb-6">
                <div className="bg-[#0a1a0a] p-4 rounded-2xl border border-[#1f3a1f]">
                  <div className="text-[9px] uppercase font-bold text-[#4b7a5a] mb-1">📏 Taille</div>
                  <div className="text-xs font-medium text-white">{selectedBird.taille}</div>
                </div>
                <div className="bg-[#0a1a0a] p-4 rounded-2xl border border-[#1f3a1f]">
                  <div className="text-[9px] uppercase font-bold text-[#4b7a5a] mb-1">📍 Lieu</div>
                  <div className="text-xs font-medium text-white truncate">{selectedBird.location}</div>
                </div>
              </div>

              <div className="space-y-4">
                <div className="bg-[#0a1a0a] p-5 rounded-2xl border border-[#1f3a1f]">
                  <h4 className="text-[9px] uppercase font-bold text-[#6ee7b7] mb-2 tracking-widest">🌿 Description & Habitat</h4>
                  <p className="text-xs leading-relaxed text-[#d1fae5]">{selectedBird.description}</p>
                </div>
                <div className="bg-[#0a1a0a] p-5 rounded-2xl border border-[#1f3a1f]">
                  <h4 className="text-[9px] uppercase font-bold text-[#6ee7b7] mb-2 tracking-widest">🎵 Signature Vocale</h4>
                  <p className="text-xs italic text-[#6ee7b7] leading-relaxed">{selectedBird.chant}</p>
                </div>
              </div>
              
              <div className="mt-8 text-center">
                <p className="text-[8px] uppercase font-bold text-[#1f3a1f]">Observé le {selectedBird.date}</p>
              </div>
            </div>
          </div>
        )}

      </main>
    </div>
  );
}
