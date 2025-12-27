<!DOCTYPE html>
<html lang="pt-PT">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no, viewport-fit=cover">
    <title>NutriScan Pro | Scanner Nutricional Inteligente</title>
    
    <!-- Frameworks e Bibliotecas Externas (Via CDN para uso com Internet) -->
    <script src="https://cdn.tailwindcss.com"></script>
    <script src="https://unpkg.com/react@18/umd/react.production.min.js"></script>
    <script src="https://unpkg.com/react-dom@18/umd/react-dom.production.min.js"></script>
    <script src="https://unpkg.com/@babel/standalone/babel.min.js"></script>
    <script src="https://unpkg.com/lucide@latest"></script>

    <style>
        @import url('https://fonts.googleapis.com/css2?family=Plus+Jakarta+Sans:wght@400;600;800&display=swap');
        
        :root {
            --brand-green: #22c55e;
            --dark-bg: #050810;
        }

        body {
            font-family: 'Plus Jakarta Sans', sans-serif;
            background-color: var(--dark-bg);
            color: #ffffff;
            margin: 0;
            -webkit-font-smoothing: antialiased;
            overscroll-behavior-y: contain;
        }

        .scanner-line {
            height: 3px;
            background: linear-gradient(90deg, transparent, var(--brand-green), transparent);
            position: absolute;
            width: 100%;
            top: 0;
            z-index: 30;
            animation: scanning 2s infinite ease-in-out;
            box-shadow: 0 0 20px var(--brand-green);
        }

        @keyframes scanning {
            0% { top: 0%; opacity: 0; }
            50% { opacity: 1; }
            100% { top: 100%; opacity: 0; }
        }

        .glass-effect {
            background: rgba(15, 23, 42, 0.8);
            backdrop-filter: blur(20px);
            border: 1px solid rgba(255, 255, 255, 0.08);
        }

        .card-shadow {
            box-shadow: 0 20px 50px rgba(0,0,0,0.5);
        }

        /* Animações de entrada */
        .fade-up {
            animation: fadeUp 0.5s ease-out forwards;
        }

        @keyframes fadeUp {
            from { opacity: 0; transform: translateY(20px); }
            to { opacity: 1; transform: translateY(0); }
        }

        input[type="file"] { display: none; }
    </style>
</head>
<body>
    <div id="root"></div>

    <script type="text/babel">
        const { useState, useRef, useEffect } = React;

        // Utilitário para ícones Lucide em tempo real
        const Icon = ({ name, size = 20, className = "" }) => {
            useEffect(() => {
                if (window.lucide) window.lucide.createIcons();
            }, [name]);
            return <i data-lucide={name} className={className} style={{ width: size, height: size }}></i>;
        };

        const App = () => {
            const [lang, setLang] = useState('pt');
            const [image, setImage] = useState(null);
            const [base64, setBase64] = useState(null);
            const [analysis, setAnalysis] = useState(null);
            const [loading, setLoading] = useState(false);
            const [error, setError] = useState(null);
            
            const camInput = useRef(null);
            const galInput = useRef(null);
            
            // Chave de API vazia - o ambiente preencherá ou o utilizador insere
            const apiKey = ""; 

            const t = {
                pt: {
                    title: "NUTRISCAN",
                    tag: "ONLINE ENGINE v5.0",
                    camera: "Tirar Foto",
                    gallery: "Abrir Galeria",
                    analyze: "Analisar Alimento",
                    processing: "A digitalizar bio-dados...",
                    reset: "Novo Scan",
                    energy: "Calorias",
                    macros: { p: "Proteína", c: "Carbs", f: "Gorduras" },
                    profiles: {
                        Geral: "Geral",
                        Diabetes: "Controlo Glicémico",
                        Hernia: "Saúde Digestiva",
                        Vigor: "Performance"
                    }
                },
                en: {
                    title: "NUTRISCAN",
                    tag: "ONLINE ENGINE v5.0",
                    camera: "Take Photo",
                    gallery: "Gallery",
                    analyze: "Analyze Food",
                    processing: "Scanning bio-data...",
                    reset: "New Scan",
                    energy: "Calories",
                    macros: { p: "Protein", c: "Carbs", f: "Fat" },
                    profiles: {
                        Geral: "General",
                        Diabetes: "Glycemic Control",
                        Hernia: "Digestive Health",
                        Vigor: "Performance"
                    }
                }
            };

            const handleFile = (e) => {
                const file = e.target.files[0];
                if (file) {
                    const reader = new FileReader();
                    reader.onloadend = () => {
                        setImage(URL.createObjectURL(file));
                        setBase64(reader.result.split(',')[1]);
                        setAnalysis(null);
                        setError(null);
                    };
                    reader.readAsDataURL(file);
                }
            };

            const runScan = async () => {
                if (!base64) return;
                setLoading(true);
                setError(null);

                // Prompt otimizado para a versão online
                const prompt = `Act as a clinical nutritionist. Analyze the food in this image.
                Provide results for 4 profiles: General, Diabetes, Hernia, and Performance.
                Respond STRICTLY in JSON format. Language: ${lang === 'pt' ? 'Portuguese' : 'English'}.
                
                JSON Schema:
                {
                  "food": "Name of Dish",
                  "cal": 0,
                  "macros": {"p": "0g", "c": "0g", "f": "0g"},
                  "profiles": [
                    {"id": "Geral", "score": "A-E", "obs": "Short clinical comment"},
                    {"id": "Diabetes", "score": "A-E", "obs": "Short clinical comment"},
                    {"id": "Hernia", "score": "A-E", "obs": "Short clinical comment"},
                    {"id": "Vigor", "score": "A-E", "obs": "Short clinical comment"}
                  ],
                  "tip": "One powerful advice"
                }`;

                try {
                    const response = await fetch(`https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash-preview-09-2025:generateContent?key=${apiKey}`, {
                        method: 'POST',
                        headers: { 'Content-Type': 'application/json' },
                        body: JSON.stringify({
                            contents: [{ parts: [{ text: prompt }, { inlineData: { mimeType: "image/png", data: base64 } }] }],
                            generationConfig: { responseMimeType: "application/json" }
                        })
                    });

                    if (!response.ok) throw new Error("API Error");

                    const data = await response.json();
                    const resultText = data.candidates[0].content.parts[0].text;
                    setAnalysis(JSON.parse(resultText));
                } catch (err) {
                    setError(lang === 'pt' ? "Falha na análise. Verifique a sua ligação à internet." : "Analysis failed. Check your internet connection.");
                } finally {
                    setLoading(false);
                }
            };

            return (
                <div className="flex flex-col min-h-screen max-w-md mx-auto relative border-x border-white/5 bg-[#050810]">
                    {/* Top Bar */}
                    <nav className="p-5 glass-effect sticky top-0 z-50 flex justify-between items-center">
                        <div className="flex items-center gap-3">
                            <div className="w-10 h-10 bg-green-500 rounded-2xl flex items-center justify-center shadow-[0_0_20px_rgba(34,197,94,0.4)]">
                                <Icon name="zap" size={22} className="text-white fill-white" />
                            </div>
                            <div>
                                <h1 className="text-lg font-extrabold italic tracking-tighter leading-none">
                                    {t[lang].title}<span className="text-green-500 ml-0.5">PRO</span>
                                </h1>
                                <p className="text-[7px] font-bold uppercase tracking-[0.4em] text-green-500/60 mt-1">{t[lang].tag}</p>
                            </div>
                        </div>
                        <button 
                            onClick={() => setLang(lang === 'pt' ? 'en' : 'pt')}
                            className="bg-slate-800/80 px-4 py-2 rounded-xl text-[10px] font-black border border-white/10 uppercase tracking-widest"
                        >
                            {lang.toUpperCase()}
                        </button>
                    </nav>

                    <main className="p-6 flex-1 space-y-6">
                        
                        {/* Scanner Area */}
                        <div className="relative aspect-square rounded-[3rem] overflow-hidden bg-slate-900/50 border border-white/5 card-shadow">
                            {!image ? (
                                <div className="absolute inset-0 flex flex-col items-center justify-center p-8 text-center">
                                    <div className="mb-8 relative">
                                        <div className="w-24 h-24 bg-green-500/10 rounded-full flex items-center justify-center border border-green-500/20">
                                            <Icon name="maximize" size={32} className="text-green-500" />
                                        </div>
                                        <div className="absolute -inset-4 border border-green-500/10 rounded-full animate-ping"></div>
                                    </div>
                                    
                                    <div className="space-y-4 w-full">
                                        <button 
                                            onClick={() => camInput.current.click()} 
                                            className="w-full bg-green-500 text-white font-black py-5 rounded-3xl text-[11px] uppercase tracking-[0.2em] flex items-center justify-center gap-3 hover:brightness-110 active:scale-95 transition-all shadow-xl shadow-green-500/20"
                                        >
                                            <Icon name="camera" size={18} /> {t[lang].camera}
                                        </button>
                                        <button 
                                            onClick={() => galInput.current.click()} 
                                            className="w-full bg-slate-800/80 text-white font-black py-5 rounded-3xl text-[11px] uppercase tracking-[0.2em] flex items-center justify-center gap-3 active:scale-95 border border-white/10"
                                        >
                                            <Icon name="image" size={18} /> {t[lang].gallery}
                                        </button>
                                    </div>
                                </div>
                            ) : (
                                <div className="h-full w-full relative group">
                                    <img src={image} className="w-full h-full object-cover transition-transform duration-700 group-hover:scale-105" />
                                    <div className="scanner-line"></div>
                                    
                                    {loading && (
                                        <div className="absolute inset-0 bg-black/80 backdrop-blur-xl flex flex-col items-center justify-center p-10 text-center">
                                            <div className="relative w-16 h-16 mb-6">
                                                <div className="absolute inset-0 border-4 border-green-500/20 rounded-full"></div>
                                                <div className="absolute inset-0 border-4 border-green-500 border-t-transparent rounded-full animate-spin"></div>
                                            </div>
                                            <p className="text-[10px] font-black uppercase tracking-[0.5em] text-green-500 animate-pulse">{t[lang].processing}</p>
                                        </div>
                                    )}

                                    {!loading && !analysis && (
                                        <div className="absolute bottom-6 left-6 right-6 flex gap-3 fade-up">
                                            <button onClick={() => {setImage(null); setAnalysis(null);}} className="p-5 bg-red-500/10 text-red-500 rounded-3xl backdrop-blur-xl border border-red-500/20 active:scale-90 transition-all">
                                                <Icon name="x" size={24} />
                                            </button>
                                            <button 
                                                onClick={runScan} 
                                                className="flex-1 bg-green-500 text-white font-black uppercase text-[11px] tracking-[0.2em] rounded-3xl shadow-2xl shadow-green-500/40 active:scale-95 transition-all flex items-center justify-center gap-3"
                                            >
                                                <Icon name="activity" size={20} /> {t[lang].analyze}
                                            </button>
                                        </div>
                                    )}
                                </div>
                            )}
                        </div>

                        {/* Error State */}
                        {error && (
                            <div className="bg-red-500/10 border border-red-500/20 p-5 rounded-3xl text-center fade-up">
                                <p className="text-xs font-bold text-red-400">{error}</p>
                            </div>
                        )}

                        {/* Results */}
                        {analysis && (
                            <div className="space-y-6 fade-up">
                                {/* Nutritional Score */}
                                <div className="bg-white rounded-[2.5rem] p-8 text-slate-950 shadow-2xl relative overflow-hidden">
                                    <div className="absolute top-0 right-0 p-6 opacity-[0.03] rotate-12">
                                        <Icon name="pie-chart" size={120} className="text-black" />
                                    </div>
                                    
                                    <div className="flex justify-between items-start mb-8">
                                        <div>
                                            <span className="text-[8px] font-black text-slate-400 uppercase tracking-widest mb-1 block">Alimento Detetado</span>
                                            <h2 className="text-3xl font-black italic uppercase tracking-tighter leading-tight">{analysis.food}</h2>
                                        </div>
                                        <div className="bg-slate-900 text-white px-5 py-3 rounded-2xl text-center shadow-lg">
                                            <span className="text-[8px] font-bold block uppercase opacity-50 mb-1">{t[lang].energy}</span>
                                            <span className="text-2xl font-black">{analysis.cal} <small className="text-[10px] opacity-40">kcal</small></span>
                                        </div>
                                    </div>
                                    
                                    <div className="grid grid-cols-3 gap-4">
                                        <MacroItem label={t[lang].macros.p} val={analysis.macros.p} />
                                        <MacroItem label={t[lang].macros.c} val={analysis.macros.c} />
                                        <MacroItem label={t[lang].macros.f} val={analysis.macros.f} />
                                    </div>
                                </div>

                                {/* Clinical Profiles */}
                                <div className="space-y-3">
                                    <p className="text-[9px] font-black text-slate-500 uppercase tracking-[0.4em] px-4">Relatório de Saúde</p>
                                    {analysis.profiles.map((p, idx) => (
                                        <div key={idx} className="glass-effect p-6 rounded-[2rem] flex flex-col gap-3 group hover:border-green-500/30 transition-all">
                                            <div className="flex justify-between items-center">
                                                <div className="flex items-center gap-3">
                                                    <div className="w-2 h-2 rounded-full bg-green-500"></div>
                                                    <span className="text-[11px] font-black uppercase tracking-widest text-slate-300">
                                                        {t[lang].profiles[p.id]}
                                                    </span>
                                                </div>
                                                <span className={`text-[11px] font-black px-3 py-1 rounded-full border ${
                                                    ['A','B'].includes(p.score) 
                                                    ? 'text-green-500 border-green-500/20 bg-green-500/5' 
                                                    : 'text-orange-500 border-orange-500/20 bg-orange-500/5'
                                                }`}>
                                                    SCORE {p.score}
                                                </span>
                                            </div>
                                            <p className="text-[12px] text-slate-400 leading-relaxed italic border-l-2 border-white/5 pl-4">
                                                {p.obs}
                                            </p>
                                        </div>
                                    ))}
                                </div>

                                {/* Verdict */}
                                <div className="bg-gradient-to-br from-green-600 to-green-700 p-8 rounded-[2.5rem] flex items-center gap-6 shadow-xl shadow-green-500/20">
                                    <div className="w-14 h-14 bg-white/20 rounded-2xl flex items-center justify-center text-white shrink-0">
                                        <Icon name="shield-check" size={32} />
                                    </div>
                                    <p className="text-xs font-bold text-white leading-relaxed italic uppercase tracking-tight">
                                        "{analysis.tip}"
                                    </p>
                                </div>

                                <button 
                                    onClick={() => {setImage(null); setAnalysis(null);}} 
                                    className="w-full py-12 text-[10px] font-black uppercase tracking-[0.6em] text-slate-700 border-2 border-dashed border-white/5 rounded-[2.5rem] hover:text-green-500 hover:border-green-500/20 transition-all active:scale-95"
                                >
                                    {t[lang].reset}
                                </button>
                            </div>
                        )}
                    </main>

                    {/* Footer */}
                    <footer className="p-12 text-center border-t border-white/5 bg-black/20">
                        <p className="text-[10px] font-black uppercase tracking-[0.3em] opacity-40">Adelino Colacama & Ana Anastâncio</p>
                        <p className="text-[7px] font-bold uppercase tracking-[0.5em] mt-3 opacity-20 italic underline decoration-green-500/30">Lunda Sul • Angola 2025</p>
                    </footer>

                    {/* Hidden Inputs */}
                    <input type="file" ref={camInput} capture="environment" onChange={handleFile} />
                    <input type="file" ref={galInput} onChange={handleFile} />
                </div>
            );
        };

        const MacroItem = ({ label, val }) => (
            <div className="bg-slate-50 p-5 rounded-2xl text-center border border-slate-100/50">
                <span className="text-[8px] font-black text-slate-400 block mb-1 uppercase tracking-tighter">{label}</span>
                <span className="text-sm font-black text-slate-900">{val}</span>
            </div>
        );

        const root = ReactDOM.createRoot(document.getElementById('root'));
        root.render(<App />);
    </script>
</body>
</html>

