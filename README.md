import React, { useState, useEffect, useRef, useMemo } from 'react';
import Navbar from './components/Navbar';
import Footer from './components/Footer';
import MusicPlayer from './components/MusicPlayer';
import { ARTISTS, RELEASES, EVENTS, PLAYLIST, TEAM, STORE_ITEMS, GALLERY_ITEMS, DIST_FEATURES } from './constants';
import { Track, Artist, Currency, StoreItem, User, CartItem, Release, GalleryItem } from './types';
import { getDemoFeedback } from './services/geminiService';
import { GoogleGenAI, LiveServerMessage, Modality } from '@google/genai';

const App: React.FC = () => {
  const [activePage, setActivePage] = useState('home');
  const [isDarkMode, setIsDarkMode] = useState(() => {
    const saved = localStorage.getItem('cme_theme');
    return saved === null ? true : saved === 'dark';
  });
  const [currentTrackIndex, setCurrentTrackIndex] = useState(0);
  const [isPlaying, setIsPlaying] = useState(false);
  const [demoLyrics, setDemoLyrics] = useState('');
  const [demoFeedback, setDemoFeedback] = useState<any>(null);
  const [isSubmitting, setIsSubmitting] = useState(false);
  const [musicFilter, setMusicFilter] = useState('all');
  const [selectedArtist, setSelectedArtist] = useState<Artist | null>(null);
  const [bookingStatus, setBookingStatus] = useState<'idle' | 'submitting' | 'success'>('idle');
  const [selectedGalleryItem, setSelectedGalleryItem] = useState<GalleryItem | null>(null);
  
  // Distribution/Payment States
  const [showDistCheckout, setShowDistCheckout] = useState(false);
  const [distCheckoutStep, setDistCheckoutStep] = useState<'plan' | 'payment' | 'processing' | 'success'>('plan');
  const [distProvider, setDistProvider] = useState<'MTN' | 'AIRTEL' | null>(null);
  const [phoneNumber, setPhoneNumber] = useState('');

  // Live API States
  const [isLiveActive, setIsLiveActive] = useState(false);
  const nextStartTimeRef = useRef(0);
  const audioContextRef = useRef<AudioContext | null>(null);
  const outputNodeRef = useRef<GainNode | null>(null);
  const sourcesRef = useRef<Set<AudioBufferSourceNode>>(new Set());

  // Store States
  const [currency, setCurrency] = useState<Currency>('UGX');
  const [storeCategory, setStoreCategory] = useState<string>('all');
  const [cart, setCart] = useState<CartItem[]>([]);
  const [cartSuccess, setCartSuccess] = useState<string | null>(null);
  const [showCheckout, setShowCheckout] = useState(false);

  // Auth States
  const [user, setUser] = useState<User>({ email: '', isLoggedIn: false });
  const [loginForm, setLoginForm] = useState({ email: '', password: '', remember: false });
  const [isRobotChecked, setIsRobotChecked] = useState(false);
  const [robotVerifying, setRobotVerifying] = useState(false);

  // Gallery State
  const [galleryFilter, setGalleryFilter] = useState<string>('all');

  // Theme constants
  const labelColor = isDarkMode ? 'text-[#D4AF37]/60' : 'text-[#D4AF37]/80';
  const subTitleColor = isDarkMode ? 'text-[#D4AF37]/40' : 'text-zinc-500';

  // Global Keyboard Listener for Closing Modals
  useEffect(() => {
    const handleKeyDown = (e: KeyboardEvent) => {
      if (e.key === 'Escape') {
        setSelectedGalleryItem(null);
        setSelectedArtist(null);
        setShowCheckout(false);
        setShowDistCheckout(false);
      }
    };
    window.addEventListener('keydown', handleKeyDown);
    return () => window.removeEventListener('keydown', handleKeyDown);
  }, []);

  // Load persistence
  useEffect(() => {
    const savedUser = localStorage.getItem('cme_user');
    if (savedUser) setUser(JSON.parse(savedUser));
    const savedCart = localStorage.getItem('cme_cart');
    if (savedCart) setCart(JSON.parse(savedCart));
    window.scrollTo({ top: 0, behavior: 'smooth' });
  }, [activePage]);

  useEffect(() => {
    localStorage.setItem('cme_cart', JSON.stringify(cart));
  }, [cart]);

  useEffect(() => {
    document.body.className = isDarkMode ? 'dark-mode' : 'light-mode';
    localStorage.setItem('cme_theme', isDarkMode ? 'dark' : 'light');
  }, [isDarkMode]);

  const currentTrack = PLAYLIST[currentTrackIndex] || null;

  const handlePlayTrack = (track: Track) => {
    const index = PLAYLIST.findIndex(t => t.id === track.id);
    if (currentTrackIndex === index) {
      setIsPlaying(!isPlaying);
    } else {
      setCurrentTrackIndex(index);
      setIsPlaying(true);
    }
  };

  const handleNextTrack = () => {
    setCurrentTrackIndex((prev) => (prev + 1) % PLAYLIST.length);
    setIsPlaying(true);
  };

  const handlePrevTrack = () => {
    setCurrentTrackIndex((prev) => (prev - 1 + PLAYLIST.length) % PLAYLIST.length);
    setIsPlaying(true);
  };

  const toggleDarkMode = () => setIsDarkMode(!isDarkMode);

  const handlePaymentSubmit = (e: React.FormEvent) => {
    e.preventDefault();
    setDistCheckoutStep('processing');
    setTimeout(() => {
      setDistCheckoutStep('success');
    }, 3000);
  };

  const openDistCheckout = () => {
    if (!user.isLoggedIn) {
      setActivePage('login');
      return;
    }
    setDistCheckoutStep('payment');
    setShowDistCheckout(true);
  };

  const startLiveAandR = async () => {
    try {
      setIsLiveActive(true);
      const ai = new GoogleGenAI({ apiKey: process.env.API_KEY });
      
      if (!audioContextRef.current) {
        audioContextRef.current = new (window.AudioContext || (window as any).webkitAudioContext)({ sampleRate: 24000 });
        outputNodeRef.current = audioContextRef.current.createGain();
        outputNodeRef.current.connect(audioContextRef.current.destination);
      }

      const stream = await navigator.mediaDevices.getUserMedia({ audio: true });
      const inputCtx = new (window.AudioContext || (window as any).webkitAudioContext)({ sampleRate: 16000 });
      
      const sessionPromise = ai.live.connect({
        model: 'gemini-2.5-flash-native-audio-preview-12-2025',
        callbacks: {
          onopen: () => {
            const source = inputCtx.createMediaStreamSource(stream);
            const scriptProcessor = inputCtx.createScriptProcessor(4096, 1, 1);
            scriptProcessor.onaudioprocess = (e) => {
              const inputData = e.inputBuffer.getChannelData(0);
              const l = inputData.length;
              const int16 = new Int16Array(l);
              for (let i = 0; i < l; i++) int16[i] = inputData[i] * 32768;
              const bytes = new Uint8Array(int16.buffer);
              let binary = '';
              for (let i = 0; i < bytes.byteLength; i++) binary += String.fromCharCode(bytes[i]);
              const base64 = btoa(binary);
              
              sessionPromise.then(s => s.sendRealtimeInput({ 
                media: { data: base64, mimeType: 'audio/pcm;rate=16000' } 
              }));
            };
            source.connect(scriptProcessor);
            scriptProcessor.connect(inputCtx.destination);
          },
          onmessage: async (msg: LiveServerMessage) => {
            if (msg.serverContent?.modelTurn?.parts?.[0]?.inlineData?.data) {
              const base64 = msg.serverContent.modelTurn.parts[0].inlineData.data;
              const binaryString = atob(base64);
              const bytes = new Uint8Array(binaryString.length);
              for (let i = 0; i < binaryString.length; i++) bytes[i] = binaryString.charCodeAt(i);
              
              const ctx = audioContextRef.current!;
              const dataInt16 = new Int16Array(bytes.buffer);
              const frameCount = dataInt16.length;
              const buffer = ctx.createBuffer(1, frameCount, 24000);
              const channelData = buffer.getChannelData(0);
              for (let i = 0; i < frameCount; i++) channelData[i] = dataInt16[i] / 32768.0;

              const source = ctx.createBufferSource();
              source.buffer = buffer;
              source.connect(outputNodeRef.current!);
              
              nextStartTimeRef.current = Math.max(nextStartTimeRef.current, ctx.currentTime);
              source.start(nextStartTimeRef.current);
              nextStartTimeRef.current += buffer.duration;
              sourcesRef.current.add(source);
            }
          },
          onerror: (e) => {
            console.error("Live Error:", e);
            setIsLiveActive(false);
          },
          onclose: () => setIsLiveActive(false)
        },
        config: {
          responseModalities: [Modality.AUDIO],
          systemInstruction: 'You are the CENTURY MUSIC EMPIRE AI A&R Terminal. You are professional, visionary, and sharp. You evaluate talent for global dominance. Keep responses short, direct, and elite.'
        }
      });
    } catch (error) {
      console.error(error);
      setIsLiveActive(false);
    }
  };

  const submitDemo = async (e: React.FormEvent) => {
    e.preventDefault();
    setIsSubmitting(true);
    setDemoFeedback(null);
    const feedback = await getDemoFeedback(demoLyrics);
    setDemoFeedback(feedback);
    setIsSubmitting(false);
  };

  const handleStudioBooking = (e: React.FormEvent) => {
    e.preventDefault();
    setBookingStatus('submitting');
    setTimeout(() => {
      setBookingStatus('success');
      setTimeout(() => setBookingStatus('idle'), 3000);
    }, 1500);
  };

  const formatPrice = (amountUGX: number, amountUSD: number) => {
    if (currency === 'UGX') return `${amountUGX.toLocaleString()} UGX`;
    return `$${amountUSD.toLocaleString()}`;
  };

  const calculateTotal = () => {
    return cart.reduce(
      (acc, item) => ({
        totalUGX: acc.totalUGX + item.priceUGX * item.quantity,
        totalUSD: acc.totalUSD + item.priceUSD * item.quantity,
      }),
      { totalUGX: 0, totalUSD: 0 }
    );
  };

  const handleBuy = (item: StoreItem) => {
    if (!user.isLoggedIn) {
      setActivePage('login');
      return;
    }
    setCart(prev => {
      const existing = prev.find(i => i.id === item.id);
      if (existing) return prev.map(i => i.id === item.id ? { ...i, quantity: i.quantity + 1 } : i);
      return [...prev, { ...item, quantity: 1 }];
    });
    setCartSuccess(item.name);
    setTimeout(() => setCartSuccess(null), 3000);
  };

  const removeFromCart = (id: string) => {
    setCart(prev => prev.filter(i => i.id !== id));
  };

  const handleLogin = (e: React.FormEvent) => {
    e.preventDefault();
    if (!isRobotChecked) return;
    const loggedUser = { email: loginForm.email, isLoggedIn: true };
    setUser(loggedUser);
    if (loginForm.remember) localStorage.setItem('cme_user', JSON.stringify(loggedUser));
    setActivePage('home');
  };

  const handleRobotCheck = () => {
    if (isRobotChecked) return;
    setRobotVerifying(true);
    setTimeout(() => {
      setRobotVerifying(false);
      setIsRobotChecked(true);
    }, 800);
  };

  const handleLogout = () => {
    setUser({ email: '', isLoggedIn: false });
    localStorage.removeItem('cme_user');
    setActivePage('home');
  };

  const groupedVault = useMemo(() => {
    const filteredReleases = musicFilter === 'all' 
      ? RELEASES 
      : RELEASES.filter(r => r.artistName === musicFilter);

    return filteredReleases.map(release => {
      const tracks = PLAYLIST.filter(t => t.artistName === release.artistName && (t.title === release.title || release.type === 'album'));
      return { ...release, tracks };
    });
  }, [musicFilter]);

  const renderContent = () => {
    switch (activePage) {
      case 'home':
        return (
          <div className="space-y-32 pb-40 page-fade-in">
            {/* HERO SECTION - REFINED FOR ORGANIZED IMPACT */}
            <section className={`relative min-h-[90vh] flex flex-col items-center justify-center overflow-hidden ${isDarkMode ? 'bg-[#050505]' : 'bg-[#fdfcf0]'}`}>
              {/* Background with higher opacity for the artist image */}
              <div className="absolute inset-0 opacity-40 md:opacity-60 bg-[url('https://api.typedream.com/v1/image/public/691d1466-419b-43d7-a5ca-069004863864_image_png')] bg-cover bg-center grayscale-0" />
              <div className={`absolute inset-0 ${isDarkMode ? 'bg-gradient-to-b from-black/80 via-black/20 to-black' : 'bg-gradient-to-b from-white/60 via-transparent to-white'}`} />
              
              <div className="relative z-10 text-center px-4 max-w-6xl mx-auto -mt-12 md:-mt-24">
                <div className={`inline-block mb-4 md:mb-8 px-6 py-2 border border-[#D4AF37]/40 ${isDarkMode ? 'bg-black/40' : 'bg-white/40 shadow-sm'} backdrop-blur-md rounded-full`}>
                  <span className="text-[10px] font-black uppercase tracking-[0.5em] text-[#D4AF37]">Empire Operational Terminal v4.0.2</span>
                </div>
                <h1 className="text-7xl md:text-[13rem] font-black uppercase tracking-tighter mb-2 italic leading-[0.85] glow-text text-[#D4AF37] scale-y-95">
                  CENTURY<br/>EMPIRE
                </h1>
                <p className={`${isDarkMode ? 'text-[#D4AF37]/80' : 'text-[#D4AF37]'} font-black uppercase tracking-[0.8em] text-[11px] md:text-sm mb-12 drop-shadow-md`}>Global Sonic Authority • Establishing Legacies</p>
                
                <div className="grid grid-cols-2 md:grid-cols-4 gap-4 max-w-5xl mx-auto">
                  {[
                    { id: 'music', label: 'THE VAULT', sub: 'Archives' },
                    { id: 'streams', label: 'STREAMS', sub: 'Imperial Audio' },
                    { id: 'store', label: 'STORE', sub: 'Imperial Supply' },
                    { id: 'gallery', label: 'GALLERY', sub: 'Imperial Ops' }
                  ].map(item => (
                    <button 
                      key={item.id}
                      onClick={() => setActivePage(item.id)}
                      className={`group p-6 md:p-10 border ${isDarkMode ? 'border-[#D4AF37]/20 bg-black/60' : 'border-[#D4AF37]/40 bg-white/80'} backdrop-blur-md hover:bg-[#D4AF37]/10 transition-all border-b-4 border-b-transparent hover:border-b-[#D4AF37] shadow-2xl overflow-hidden relative`}
                    >
                      <span className={`block ${subTitleColor} font-black text-[9px] uppercase tracking-widest mb-3 relative z-10`}>{item.sub}</span>
                      <span className="block text-lg md:text-xl font-black italic text-[#D4AF37] group-hover:translate-x-1 transition-transform relative z-10">{item.label}</span>
                      <div className="absolute inset-0 bg-[#D4AF37]/5 translate-y-full group-hover:translate-y-0 transition-transform duration-500" />
                    </button>
                  ))}
                </div>
              </div>
            </section>

            <div className="bg-[#D4AF37] py-4 overflow-hidden -rotate-1 shadow-2xl relative z-10">
              <div className="flex whitespace-nowrap animate-marquee">
                {[...Array(10)].map((_, i) => (
                  <span key={i} className="text-black font-black uppercase italic text-2xl mx-16">CENTURY MUSIC EMPIRE • ACTIVE • KIM C UG • EMPIRE • </span>
                ))}
              </div>
            </div>

            <section className="max-w-7xl mx-auto px-4 grid grid-cols-1 lg:grid-cols-2 gap-20 items-center">
               <div className={`relative group overflow-hidden border ${isDarkMode ? 'border-[#D4AF37]/10' : 'border-[#D4AF37]/30'} aspect-square shadow-2xl`}>
                  <img src={ARTISTS[0].imageUrl} className="w-full h-full object-cover grayscale opacity-40 group-hover:grayscale-0 group-hover:opacity-100 transition-all duration-[2s] scale-100 group-hover:scale-110" alt="Kim Cug" />
                  <div className={`absolute inset-0 ${isDarkMode ? 'bg-gradient-to-t from-black via-transparent' : 'bg-gradient-to-t from-white/40 via-transparent'}`} />
                  <div className="absolute bottom-12 left-12">
                     <span className="bg-[#D4AF37] text-black px-4 py-1 font-black text-[10px] uppercase tracking-widest mb-4 inline-block shadow-lg">Flagship Talent</span>
                     <h3 className="text-7xl font-black italic text-[#D4AF37] uppercase glow-text">KIM C UG</h3>
                  </div>
               </div>
               <div className="space-y-10">
                  <h2 className={`${labelColor} font-black uppercase tracking-[0.6em] text-xs`}>Empire Manifest</h2>
                  <p className="text-4xl md:text-5xl font-black text-[#D4AF37] italic leading-tight uppercase">We don't just release music. We establish eternal frequencies.</p>
                  <p className={`${isDarkMode ? 'text-zinc-500' : 'text-zinc-600'} text-lg leading-relaxed`}>Through precision engineering and global distribution, Century Music Empire provides the framework for artists to transcend the temporary and become monumental.</p>
                  <div className="grid grid-cols-2 gap-4">
                     <div className={`p-8 border border-[#D4AF37]/10 ${isDarkMode ? 'bg-zinc-900/40' : 'bg-white/80 shadow-md'}`}>
                        <span className="block text-4xl font-black italic text-[#D4AF37] mb-2">124M+</span>
                        <span className="text-[10px] font-black text-zinc-600 uppercase tracking-widest">Global Streams</span>
                     </div>
                     <div className={`p-8 border border-[#D4AF37]/10 ${isDarkMode ? 'bg-zinc-900/40' : 'bg-white/80 shadow-md'}`}>
                        <span className="block text-4xl font-black italic text-[#D4AF37] mb-2">180+</span>
                        <span className="text-[10px] font-black text-zinc-600 uppercase tracking-widest">Territories</span>
                     </div>
                  </div>
                  <button onClick={() => setActivePage('artists')} className={`w-full py-6 border border-[#D4AF37]/20 text-[#D4AF37] font-black uppercase text-xs tracking-widest hover:bg-[#D4AF37] hover:text-black transition-all shadow-xl`}>Explore the Roster</button>
               </div>
            </section>
          </div>
        );
      case 'streams':
        return (
          <div className="max-w-7xl mx-auto px-4 py-32 page-fade-in">
            <div className={`mb-24 border-b ${isDarkMode ? 'border-white/10' : 'border-zinc-200'} pb-16 text-center`}>
                <h1 className="text-7xl md:text-9xl font-black uppercase mb-6 italic text-[#D4AF37] leading-none glow-text">IMPERIAL<br/>STREAMS</h1>
                <p className={`${subTitleColor} font-black uppercase tracking-[0.6em] text-[10px]`}>Global Sonic Presence • Low-Latency Audio v4.0</p>
            </div>

            <div className="grid grid-cols-1 lg:grid-cols-2 gap-16">
              <div className={`p-8 border ${isDarkMode ? 'bg-zinc-950 border-[#D4AF37]/10' : 'bg-white border-[#D4AF37]/20'} shadow-2xl space-y-6`}>
                <div className="flex justify-between items-center">
                   <h3 className="text-xl font-black italic text-[#D4AF37] uppercase">Spotify Official</h3>
                   <span className="text-green-500 font-black text-[10px] uppercase tracking-widest">Verified Channel</span>
                </div>
                <div className="aspect-video w-full rounded-xl overflow-hidden bg-black shadow-inner">
                   <iframe 
                    style={{ borderRadius: '12px' }} 
                    src="https://open.spotify.com/embed/artist/37q67X98jP5e5pI3S9W4S4?utm_source=generator&theme=0" 
                    width="100%" 
                    height="352" 
                    frameBorder="0" 
                    allow="autoplay; clipboard-write; encrypted-media; fullscreen; picture-in-picture" 
                    loading="lazy"
                   ></iframe>
                </div>
              </div>

              <div className={`p-8 border ${isDarkMode ? 'bg-zinc-950 border-[#D4AF37]/10' : 'bg-white border-[#D4AF37]/20'} shadow-2xl space-y-6`}>
                <div className="flex justify-between items-center">
                   <h3 className="text-xl font-black italic text-[#D4AF37] uppercase">Audiomack Global</h3>
                   <span className="text-orange-500 font-black text-[10px] uppercase tracking-widest">Top Trending</span>
                </div>
                <div className="aspect-video w-full rounded-xl overflow-hidden bg-black shadow-inner">
                   <iframe 
                    src="https://audiomack.com/embed/kim-c-ug/song/i-propose-to-you?background=1" 
                    scrolling="no" 
                    width="100%" 
                    height="252" 
                    frameBorder="0"
                   ></iframe>
                </div>
              </div>

              <div className={`p-8 border ${isDarkMode ? 'bg-zinc-950 border-[#D4AF37]/10' : 'bg-white border-[#D4AF37]/20'} shadow-2xl space-y-6 lg:col-span-2`}>
                <div className="flex justify-between items-center">
                   <h3 className="text-xl font-black italic text-[#D4AF37] uppercase">Imperial Video Archives</h3>
                   <span className="text-red-600 font-black text-[10px] uppercase tracking-widest">4K Cinematic</span>
                </div>
                <div className="aspect-video w-full rounded-xl overflow-hidden bg-black shadow-2xl">
                   <iframe 
                    width="100%" 
                    height="100%" 
                    src="https://www.youtube.com/embed/videoseries?list=PLy_S6o_p2E_4A48S-1j5N6R_fM_Y8p7O6" 
                    title="Imperial YouTube Terminal" 
                    frameBorder="0" 
                    allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" 
                    allowFullScreen
                   ></iframe>
                </div>
              </div>
            </div>
          </div>
        );
      case 'distribution':
        return (
          <div className="max-w-7xl mx-auto px-4 py-32 page-fade-in">
            <div className={`mb-24 border-b ${isDarkMode ? 'border-white/10' : 'border-zinc-200'} pb-16 text-center`}>
                <h1 className="text-7xl md:text-9xl font-black uppercase mb-6 italic text-[#D4AF37] leading-none glow-text">IMPERIAL<br/>DISTRIBUTION</h1>
                <p className={`${subTitleColor} font-black uppercase tracking-[0.6em] text-[10px]`}>Global Digital Deployment v4.0 • 150+ Platforms</p>
            </div>

            <div className="grid grid-cols-1 lg:grid-cols-2 gap-12 mb-32">
              <div className={`group p-12 border ${isDarkMode ? 'bg-zinc-950 border-[#D4AF37]/20' : 'bg-white border-[#D4AF37]/40'} shadow-2xl relative overflow-hidden flex flex-col justify-between`}>
                <div className="absolute top-0 right-0 w-32 h-32 bg-[#D4AF37]/5 -mr-16 -mt-16 rounded-full group-hover:scale-150 transition-transform duration-700"></div>
                <div>
                   <span className="bg-[#D4AF37] text-black px-4 py-1 text-[10px] font-black uppercase tracking-widest mb-10 inline-block shadow-lg">ELITE PROTOCOL</span>
                   <h3 className="text-5xl font-black italic text-[#D4AF37] mb-6">50,000 UGX</h3>
                   <p className={`${isDarkMode ? 'text-white' : 'text-zinc-800'} text-xl italic font-black uppercase mb-10 tracking-widest`}>PER RELEASE / KEEP 100% ROYALTIES</p>
                   <ul className={`space-y-6 ${isDarkMode ? 'text-zinc-500' : 'text-zinc-600'} text-sm uppercase font-black tracking-widest mb-12`}>
                      <li className="flex items-center"><span className="w-2 h-2 bg-[#D4AF37] mr-4"></span>150+ Streaming Platforms Reach</li>
                      <li className="flex items-center"><span className="w-2 h-2 bg-[#D4AF37] mr-4"></span>100% Artist Revenue Retention</li>
                      <li className="flex items-center"><span className="w-2 h-2 bg-[#D4AF37] mr-4"></span>Imperial Dashboard Analytics</li>
                      <li className="flex items-center"><span className="w-2 h-2 bg-[#D4AF37] mr-4"></span>Priority Global DSP Deployment</li>
                   </ul>
                </div>
                <button onClick={openDistCheckout} className="w-full py-7 bg-[#D4AF37] text-black font-black uppercase text-xs tracking-[0.4em] shadow-xl hover:scale-[1.02] transition-all">INITIATE DEPLOYMENT (MM)</button>
              </div>

              <div className={`group p-12 border ${isDarkMode ? 'bg-zinc-950 border-white/5' : 'bg-white border-zinc-200'} shadow-2xl relative overflow-hidden flex flex-col justify-between`}>
                <div className="absolute top-0 right-0 w-32 h-32 bg-white/5 -mr-16 -mt-16 rounded-full group-hover:scale-150 transition-transform duration-700"></div>
                <div>
                   <span className="bg-zinc-800 text-white px-4 py-1 text-[10px] font-black uppercase tracking-widest mb-10 inline-block shadow-lg">PARTNERSHIP PROTOCOL</span>
                   <h3 className="text-5xl font-black italic text-[#D4AF37] mb-6">FREE</h3>
                   <p className={`${isDarkMode ? 'text-white' : 'text-zinc-800'} text-xl italic font-black uppercase mb-10 tracking-widest`}>85% ARTIST / 15% LABEL SHARE</p>
                   <ul className={`space-y-6 ${isDarkMode ? 'text-zinc-500' : 'text-zinc-600'} text-sm uppercase font-black tracking-widest mb-12`}>
                      <li className="flex items-center"><span className="w-2 h-2 bg-zinc-700 mr-4"></span>150+ Platforms Reach</li>
                      <li className="flex items-center"><span className="w-2 h-2 bg-zinc-700 mr-4"></span>Label Strategic Promotion Support</li>
                      <li className="flex items-center"><span className="w-2 h-2 bg-zinc-700 mr-4"></span>Zero Upfront Deployment Fees</li>
                      <li className="flex items-center"><span className="w-2 h-2 bg-zinc-700 mr-4"></span>Imperial Sync & Media Opportunities</li>
                   </ul>
                </div>
                <button onClick={() => setActivePage('demo')} className="w-full py-7 border border-[#D4AF37]/20 text-[#D4AF37] font-black uppercase text-xs tracking-[0.4em] shadow-md hover:bg-[#D4AF37] hover:text-black transition-all">REQUEST PARTNERSHIP</button>
              </div>
            </div>

            <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-4 gap-8">
               {DIST_FEATURES.map(feat => (
                 <div key={feat.id} className={`p-10 border ${isDarkMode ? 'border-white/5 bg-zinc-900/20' : 'border-zinc-100 bg-white'} shadow-xl text-center`}>
                    <div className="text-5xl mb-8">{feat.icon}</div>
                    <h4 className="text-[#D4AF37] font-black uppercase italic mb-4">{feat.title}</h4>
                    <p className={`${isDarkMode ? 'text-zinc-500' : 'text-zinc-600'} text-xs leading-relaxed`}>{feat.description}</p>
                 </div>
               ))}
            </div>
          </div>
        );
      case 'music':
        return (
          <div className="max-w-7xl mx-auto px-4 py-32 page-fade-in">
            <div className={`mb-24 flex flex-col md:flex-row md:items-end justify-between border-b ${isDarkMode ? 'border-white/10' : 'border-zinc-200'} pb-16 gap-10`}>
               <div>
                  <h1 className="text-7xl md:text-9xl font-black uppercase italic text-[#D4AF37] leading-none glow-text">THE VAULT</h1>
                  <p className={`${subTitleColor} font-black uppercase tracking-[0.5em] text-[10px] mt-6`}>Imperial Audio Archive x Compiled 2025</p>
               </div>
               <div className="flex flex-wrap gap-4">
                  {['all', 'KIM C UG', 'LUNA VOID'].map(filter => (
                    <button 
                      key={filter}
                      onClick={() => setMusicFilter(filter)} 
                      className={`px-8 py-3 text-[10px] font-black uppercase tracking-widest border transition-all shadow-md ${musicFilter === filter ? 'bg-[#D4AF37] text-black border-[#D4AF37]' : isDarkMode ? 'border-[#D4AF37]/10 text-zinc-500 hover:border-[#D4AF37]/40' : 'border-[#D4AF37]/40 text-zinc-600 bg-white hover:bg-zinc-50'}`}
                    >
                      {filter}
                    </button>
                  ))}
               </div>
            </div>
            
            <div className="space-y-40">
              {groupedVault.map((release) => (
                <div key={release.id} className="grid grid-cols-1 lg:grid-cols-12 gap-16 items-start">
                  <div className="lg:col-span-5 sticky top-40">
                    <div className={`relative aspect-square overflow-hidden border ${isDarkMode ? 'border-[#D4AF37]/10' : 'border-[#D4AF37]/30'} shadow-2xl group`}>
                      <img src={release.coverUrl} className="w-full h-full object-cover grayscale transition-all duration-700 group-hover:grayscale-0 group-hover:scale-105" alt={release.title} />
                      <div className="absolute top-8 left-8">
                        <span className="bg-[#D4AF37] text-black px-5 py-2 text-[10px] font-black uppercase italic tracking-widest shadow-2xl">{release.type}</span>
                      </div>
                    </div>
                    <div className="mt-10">
                       <h3 className="text-5xl font-black uppercase italic text-[#D4AF37] mb-2">{release.title}</h3>
                       <p className={`${labelColor} font-black uppercase tracking-[0.4em] text-sm`}>{release.artistName}</p>
                       <p className="text-zinc-700 font-mono text-[10px] mt-6 uppercase">Timestamp: {release.releaseDate}</p>
                    </div>
                  </div>

                  <div className="lg:col-span-7">
                    <div className={`mb-6 px-6 ${isDarkMode ? 'text-[#D4AF37]/30' : 'text-[#D4AF37]/60'} font-black uppercase text-[9px] tracking-[0.5em] flex justify-between border-b ${isDarkMode ? 'border-white/5' : 'border-zinc-100'} pb-4`}>
                       <span>Index / Frequency</span>
                       <span>Duration</span>
                    </div>
                    <div className="space-y-1">
                      {release.tracks.map((track, i) => (
                        <div 
                          key={track.id} 
                          className={`flex items-center justify-between p-7 border transition-all cursor-pointer group shadow-sm ${currentTrack?.id === track.id ? (isDarkMode ? 'bg-zinc-900 border-[#D4AF37]/20' : 'bg-white border-[#D4AF37]/40') : (isDarkMode ? 'border-white/5 hover:bg-zinc-900/50' : 'border-zinc-100 hover:bg-white bg-zinc-50/30')}`} 
                          onClick={() => handlePlayTrack(track)}
                        >
                          <div className="flex items-center space-x-8">
                            <span className={`${subTitleColor} font-black text-xs font-mono`}>{String(i + 1).padStart(2, '0')}</span>
                            <div>
                              <h4 className="text-[#D4AF37] font-black text-2xl uppercase italic leading-none mb-1 group-hover:tracking-wider transition-all">
                                {track.title}
                              </h4>
                              {currentTrack?.id === track.id && isPlaying && (
                                <div className="flex items-center mt-3 space-x-1.5">
                                  <div className="w-1 h-3 bg-[#D4AF37] animate-pulse" />
                                  <div className="w-1 h-5 bg-[#D4AF37] animate-pulse delay-75" />
                                  <div className="w-1 h-3 bg-[#D4AF37] animate-pulse" />
                                </div>
                              )}
                            </div>
                          </div>
                          <div className="flex items-center space-x-10">
                             <span className="text-zinc-500 font-mono text-sm">{track.duration}</span>
                             <div className={`w-12 h-12 border ${isDarkMode ? 'border-[#D4AF37]/10' : 'border-[#D4AF37]/40'} flex items-center justify-center rounded-full group-hover:bg-[#D4AF37] group-hover:text-black transition-all`}>
                               <svg className="w-5 h-5" fill="currentColor" viewBox="0 0 20 20"><path fillRule="evenodd" d="M10 18a8 8 0 100-16 8 8 0 000 16zM9.555 7.168A1 1 0 008 8v4a1 1 0 001.555.832l3-2a1 1 0 000-1.664l-3-2z" clipRule="evenodd" /></svg>
                             </div>
                          </div>
                        </div>
                      ))}
                    </div>
                  </div>
                </div>
              ))}
            </div>
          </div>
        );
      case 'gallery':
        const filteredGallery = galleryFilter === 'all' ? GALLERY_ITEMS : GALLERY_ITEMS.filter(item => item.category === galleryFilter);
        return (
          <div className="max-w-7xl mx-auto px-4 py-32 page-fade-in">
            <div className={`mb-24 flex flex-col md:flex-row md:items-end justify-between border-b ${isDarkMode ? 'border-white/10' : 'border-zinc-200'} pb-16 gap-10`}>
               <div>
                  <h1 className="text-7xl md:text-9xl font-black uppercase italic text-[#D4AF37] leading-none glow-text">GALLERY</h1>
                  <p className={`${subTitleColor} font-black uppercase tracking-[0.5em] text-[10px] mt-6`}>Imperial Imagery & Artifacts x Compiled 2025</p>
               </div>
               <div className="flex flex-wrap gap-4">
                  {['all', 'studio', 'event', 'lifestyle'].map(filter => (
                    <button 
                      key={filter}
                      onClick={() => setGalleryFilter(filter)} 
                      className={`px-8 py-3 text-[10px] font-black uppercase tracking-widest border transition-all shadow-md ${galleryFilter === filter ? 'bg-[#D4AF37] text-black border-[#D4AF37]' : isDarkMode ? 'border-[#D4AF37]/10 text-zinc-500 hover:border-[#D4AF37]/40' : 'border-[#D4AF37]/40 text-zinc-600 bg-white hover:bg-zinc-50'}`}
                    >
                      {filter}
                    </button>
                  ))}
               </div>
            </div>

            <div className="grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-3 gap-8">
              {filteredGallery.map((item) => (
                <div 
                  key={item.id} 
                  className={`group relative aspect-[4/5] overflow-hidden border ${isDarkMode ? 'border-white/5 bg-zinc-950' : 'border-zinc-100 bg-white'} cursor-pointer shadow-xl transition-all duration-500 hover:border-[#D4AF37]/30 hover:scale-[1.02]`}
                  onClick={() => setSelectedGalleryItem(item)}
                >
                  <img src={item.imageUrl} className={`w-full h-full object-cover transition-all duration-1000 ${isDarkMode ? 'grayscale opacity-40 group-hover:grayscale-0 group-hover:opacity-100' : 'grayscale-0 opacity-100'} group-hover:scale-110`} alt={item.title} />
                  
                  <div className={`absolute inset-0 transition-opacity duration-500 ${isDarkMode ? 'bg-gradient-to-t from-black via-black/20 to-transparent opacity-80 group-hover:opacity-40' : 'bg-gradient-to-t from-[#D4AF37]/30 via-transparent to-transparent opacity-0 group-hover:opacity-100'}`} />
                  
                  <div className={`absolute bottom-8 left-8 translate-y-4 group-hover:translate-y-0 transition-all duration-500 opacity-0 group-hover:opacity-100`}>
                    <span className={`${isDarkMode ? 'text-[#D4AF37]/60' : 'text-[#D4AF37]'} font-black uppercase text-[8px] tracking-[0.5em] mb-2 block`}>{item.category}</span>
                    <h3 className={`${isDarkMode ? 'text-[#D4AF37]' : 'text-white'} font-black uppercase italic text-xl leading-none drop-shadow-lg`}>{item.title}</h3>
                  </div>
                </div>
              ))}
            </div>
          </div>
        );
      case 'studio':
        return (
          <div className="max-w-7xl mx-auto px-4 py-32 page-fade-in">
            <div className="grid grid-cols-1 lg:grid-cols-2 gap-32 mb-40">
              <div className="space-y-12">
                <div>
                  <h1 className="text-8xl font-black uppercase italic mb-6 leading-none text-[#D4AF37] glow-text">THE SOUND BOX RECORDS</h1>
                  <p className={`${isDarkMode ? 'text-zinc-400' : 'text-zinc-600'} text-2xl italic font-light leading-relaxed`}>PRODUCER KIM C PRESENTS: High-fidelity engineering for the global elite. Precision recorded at Konge Poster Road.</p>
                </div>
                
                <div className={`relative group overflow-hidden border ${isDarkMode ? 'border-[#D4AF37]/20' : 'border-[#D4AF37]/40'} shadow-2xl`}>
                   <img src="https://api.typedream.com/v1/image/public/2b16e459-f831-4191-8848-03886b361a7a_image_jpg" className="w-full h-full object-cover group-hover:scale-105 transition-transform duration-[2s]" alt="Sound Box Promo" />
                   <div className={`absolute inset-0 ${isDarkMode ? 'bg-gradient-to-t from-black via-transparent' : 'bg-gradient-to-t from-white/20 via-transparent'} opacity-40`} />
                   <div className="absolute top-10 right-10 flex flex-col items-end">
                      <span className="bg-[#D4AF37] text-black px-6 py-2 font-black uppercase italic text-xs shadow-2xl">PROMOTIONAL FEE ACTIVE</span>
                   </div>
                </div>

                <div className="grid grid-cols-1 md:grid-cols-2 gap-4">
                  {[
                    { label: 'MUSIC PRODUCTION', val: 'Studio A Engineering' },
                    { label: 'VIDEO SHOOTING', val: 'Cinematic Deployment' },
                    { label: 'MUSIC PROMOTION', val: 'Global DSP Reach' },
                    { label: 'BOOK SESSION', val: '+256 702 838 224' }
                  ].map(spec => (
                    <div key={spec.label} className={`p-8 border border-[#D4AF37]/10 flex flex-col hover:border-[#D4AF37]/60 hover:scale-[1.03] active:scale-95 transition-all duration-300 shadow-lg cursor-default ${isDarkMode ? 'bg-zinc-950' : 'bg-white'}`}>
                       <span className={`${subTitleColor} font-black uppercase text-[9px] tracking-[0.3em] mb-3`}>{spec.label}</span>
                       <span className="text-[#D4AF37] font-black italic uppercase text-lg">{spec.val}</span>
                    </div>
                  ))}
                </div>

                <div className={`mt-12 p-10 border border-[#D4AF37]/20 ${isDarkMode ? 'bg-[#D4AF37]/5' : 'bg-white shadow-xl'} flex flex-col sm:flex-row items-center justify-between group gap-8 transition-transform duration-500 hover:border-[#D4AF37]/40`}>
                   <div className="text-center sm:text-left">
                      <span className={`${subTitleColor} font-black uppercase text-[9px] tracking-[0.4em] mb-3 block`}>Imperial Asset Library</span>
                      <span className="text-[#D4AF37] font-black italic uppercase text-2xl leading-none">CME PRODUCER KIT v1.0</span>
                      <p className={`${isDarkMode ? 'text-zinc-500' : 'text-zinc-400'} text-[10px] mt-4 uppercase tracking-widest font-bold`}>44.1kHz • 24-bit • Royalty Free Samples</p>
                   </div>
                   <button 
                    onClick={() => {alert("Accessing Imperial Sound Archives... Transfer initiating."); window.open('#', '_blank');}}
                    className="w-full sm:w-auto px-12 py-5 bg-[#D4AF37] text-black font-black uppercase text-[10px] tracking-[0.4em] shadow-2xl hover:scale-110 hover:brightness-110 active:scale-95 transition-all flex items-center justify-center gap-4"
                   >
                      <svg className="w-5 h-5" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path strokeLinecap="round" strokeLinejoin="round" strokeWidth={3} d="M4 16v1a3 3 0 003 3h10a3 3 0 003-3v-1m-4-4l-4 4m0 0l-4-4m4 4V4"/></svg>
                      DOWNLOAD KIT
                   </button>
                </div>
              </div>

              <div className={`p-12 md:p-16 border border-[#D4AF37]/20 shadow-2xl relative flex flex-col justify-between ${isDarkMode ? 'bg-zinc-950' : 'bg-white'}`}>
                <div>
                  <div className="absolute -top-4 -right-4 bg-[#D4AF37] text-black px-5 py-2 text-[10px] font-black uppercase italic shadow-lg">Direct Dispatch</div>
                  <h3 className="text-4xl font-black uppercase italic mb-12 text-[#D4AF37]">SECURE TERMINAL</h3>
                  <p className={`${isDarkMode ? 'text-zinc-500' : 'text-zinc-600'} mb-10 text-sm font-light italic`}>For immediate session allocation at Konge Poster Road, communicate via WhatsApp or the secure form below.</p>
                  
                  <div className="mb-12 p-8 border border-green-500/20 bg-green-500/5 flex items-center justify-between group cursor-pointer hover:bg-green-500/10 hover:scale-[1.02] active:scale-[0.98] transition-all shadow-sm" onClick={() => window.open('https://wa.me/256702838224')}>
                     <div className="flex items-center space-x-6">
                        <div className="w-12 h-12 bg-green-500 flex items-center justify-center rounded-lg shadow-lg group-hover:scale-110 transition-transform">
                           <svg className="w-8 h-8 text-white" fill="currentColor" viewBox="0 0 24 24"><path d="M17.472 14.382c-.297-.149-1.758-.867-2.03-.967-.273-.099-.471-.148-.67.15-.197.297-.767.966-.94 1.164-.173.199-.347.223-.644.075-.297-.15-1.255-.463-2.39-1.475-.883-.788-1.48-1.761-1.653-2.059-.173-.297-.018-.458.13-.606.134-.133.298-.347.446-.52.149-.174.198-.298.298-.497.099-.198.05-.371-.025-.52-.075-.149-.669-1.612-.916-2.207-.242-.579-.487-.5-.669-.51-.173-.008-.371-.01-.57-.01-.198 0-.52.074-.792.372-.272.297-1.04 1.016-1.04 2.479 0 1.462 1.065 2.875 1.213 3.074.149.198 2.096 3.2 5.077 4.487.709.306 1.262.489 1.694.625.712.227 1.36.195 1.871.118.571-.085 1.758-.719 2.006-1.413.248-.694.248-1.289.173-1.413-.074-.124-.272-.198-.57-.347m-5.421 7.403h-.004a9.87 9.87 0 01-5.031-1.378l-.361-.214-3.741.982.998-3.648-.235-.374a9.86 9.86 0 01-1.51-5.26c.001-5.45 4.436-9.884 9.888-9.884 2.64 0 5.122 1.03 6.988 2.898a9.825 9.825 0 012.893 6.994c-.003 5.45-4.437 9.884-9.885 9.884m8.413-18.297A11.815 11.815 0 0012.05 0C5.495 0 .16 5.335.157 11.892c0 2.096.547 4.142 1.588 5.945L.057 24l6.305-1.654a11.882 11.882 0 005.683 1.448h.005c6.554 0 11.89-5.335 11.893-11.893a11.821 11.821 0 00-3.48-8.413z"/></svg>
                        </div>
                        <div>
                           <span className={`block ${isDarkMode ? 'text-zinc-500' : 'text-zinc-400'} text-[10px] font-black uppercase tracking-widest`}>WhatsApp Direct</span>
                           <span className={`${isDarkMode ? 'text-white' : 'text-zinc-800'} font-black italic text-xl`}>0702838224</span>
                        </div>
                     </div>
                     <div className="text-green-500 opacity-0 group-hover:opacity-100 group-hover:translate-x-2 transition-all">
                        <svg className="w-8 h-8" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path strokeLinecap="round" strokeLinejoin="round" strokeWidth={2} d="M14 5l7 7m0 0l-7 7m7-7H3"/></svg>
                     </div>
                  </div>

                  <form onSubmit={handleStudioBooking} className="space-y-8">
                    <div className="space-y-2">
                      <label className={`text-[10px] font-black uppercase ${isDarkMode ? 'text-[#D4AF37]/60' : 'text-[#D4AF37]'} tracking-widest ml-1`}>Operator ID / Alias</label>
                      <input required type="text" className={`w-full ${isDarkMode ? 'bg-black text-white' : 'bg-zinc-50 text-zinc-900'} border ${isDarkMode ? 'border-white/10' : 'border-zinc-200'} p-5 outline-none focus:border-[#D4AF37] transition-all italic text-lg shadow-inner`} placeholder="IDENTIFY YOURSELF" />
                    </div>
                    <div className="space-y-2">
                      <label className={`text-[10px] font-black uppercase ${isDarkMode ? 'text-[#D4AF37]/60' : 'text-[#D4AF37]'} tracking-widest ml-1`}>Service Type</label>
                      <select className={`w-full ${isDarkMode ? 'bg-black' : 'bg-zinc-50'} border ${isDarkMode ? 'border-white/10' : 'border-zinc-200'} p-5 text-[#D4AF37] outline-none focus:border-[#D4AF37] transition-all italic text-lg shadow-inner`}>
                         <option>MUSIC PRODUCTION</option>
                         <option>VIDEO SHOOTING</option>
                         <option>MUSIC PROMOTION</option>
                      </select>
                    </div>
                    <button type="submit" disabled={bookingStatus !== 'idle'} className={`w-full py-7 font-black uppercase tracking-[0.4em] text-xs transition-all shadow-xl hover:brightness-110 active:scale-95 disabled:opacity-50 ${bookingStatus === 'success' ? 'bg-green-600 text-white' : 'bg-[#D4AF37] text-black'}`}>
                      {bookingStatus === 'idle' ? 'ESTABLISH SESSION' : bookingStatus === 'submitting' ? 'SYNCHRONIZING...' : 'BROADCAST SUCCESS'}
                    </button>
                  </form>
                </div>
                <div className={`mt-12 text-center text-[9px] font-black uppercase tracking-[0.5em] ${subTitleColor}`}>Imperial Technical Division v4.0.2</div>
              </div>
            </div>

            <div className="space-y-8">
               <div className="flex justify-between items-end">
                  <h3 className="text-3xl font-black italic uppercase text-[#D4AF37] glow-text">TERMINAL LOCATION</h3>
                  <span className={`${subTitleColor} font-black uppercase text-[10px] tracking-widest`}>Konge Poster Road, Kampala</span>
               </div>
               <div className={`aspect-[21/9] w-full border ${isDarkMode ? 'border-[#D4AF37]/10 grayscale invert opacity-30 hover:grayscale-0 hover:invert-0' : 'border-[#D4AF37]/40 grayscale opacity-40 hover:grayscale-0 hover:opacity-100'} hover:opacity-100 transition-all duration-[2s] overflow-hidden shadow-2xl`}>
                  <iframe 
                   src="https://www.google.com/maps/embed?pb=!1m18!1m12!1m3!1d15959.030573934444!2d32.58252!3d0.313611!2m3!1f0!2f0!3f0!3m2!1i1024!2i768!4f13.1!3m3!1m2!1s0x177dbc0917032743%3A0xf63908f58359b360!2sKampala%2C%20Uganda!5e0!3m2!1sen!2sus!4v1716500000000!5m2!1sen!2sus" 
                   width="100%" height="100%" loading="lazy" className="border-0 h-full w-full"
                  ></iframe>
               </div>
            </div>
          </div>
        );
      case 'store':
        const filteredStoreItems = storeCategory === 'all' ? STORE_ITEMS : STORE_ITEMS.filter(item => item.category === storeCategory);
        return (
          <div className="max-w-7xl mx-auto px-4 py-32 page-fade-in">
            <div className={`flex flex-col md:flex-row justify-between items-end mb-24 gap-12 border-b ${isDarkMode ? 'border-white/10' : 'border-zinc-200'} pb-16`}>
              <div>
                <h1 className="text-7xl md:text-9xl font-black uppercase italic leading-none text-[#D4AF37] glow-text">IMPERIAL<br/>STORE</h1>
                <p className={`${subTitleColor} font-black uppercase tracking-[0.5em] text-[10px] mt-6`}>Official Logistics & Merchandising x v3.0</p>
              </div>
              <div className="flex items-center space-x-6">
                <button onClick={() => setCurrency(currency === 'UGX' ? 'USD' : 'UGX')} className={`px-10 py-4 border border-[#D4AF37]/20 text-[10px] font-black uppercase text-[#D4AF37] hover:border-[#D4AF37] transition-all tracking-widest shadow-md ${isDarkMode ? 'bg-zinc-900' : 'bg-white'}`}>CURRENCY: {currency}</button>
                {cart.length > 0 && <button onClick={() => setShowCheckout(true)} className="px-12 py-4 bg-[#D4AF37] text-black text-[10px] font-black uppercase tracking-widest shadow-2xl animate-pulse">VAULT ({cart.length})</button>}
              </div>
            </div>

            <div className="flex flex-wrap gap-4 mb-20">
              {['all', 'beats', 'merch', 'gear'].map(cat => (
                <button key={cat} onClick={() => setStoreCategory(cat)} className={`px-12 py-3 text-[10px] font-black uppercase tracking-widest border transition-all shadow-sm ${storeCategory === cat ? 'bg-[#D4AF37] text-black border-[#D4AF37]' : isDarkMode ? 'text-zinc-600 border-white/10 hover:border-[#D4AF37]/40' : 'text-zinc-600 border-zinc-200 bg-white hover:bg-zinc-50'}`}>{cat}</button>
              ))}
            </div>

            <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-10">
              {filteredStoreItems.map(item => (
                <div key={item.id} className={`group border border-[#D4AF37]/5 overflow-hidden flex flex-col hover:border-[#D4AF37]/30 transition-all shadow-2xl ${isDarkMode ? 'bg-zinc-950' : 'bg-white'}`}>
                  <div className={`aspect-square relative overflow-hidden ${isDarkMode ? 'bg-zinc-900' : 'bg-zinc-100'}`}>
                    <img src={item.imageUrl} className="w-full h-full object-cover grayscale transition-all duration-[1s] group-hover:grayscale-0 group-hover:scale-105" alt={item.name} />
                    <span className="absolute top-8 left-8 bg-[#D4AF37] px-4 py-2 text-[10px] font-black text-black uppercase border border-black/10 tracking-widest shadow-lg">{item.category}</span>
                  </div>
                  <div className="p-12 flex flex-col flex-grow">
                    <h3 className="text-3xl font-black uppercase italic mb-6 text-[#D4AF37] leading-none">{item.name}</h3>
                    <p className={`${isDarkMode ? 'text-zinc-500' : 'text-zinc-600'} text-base italic font-light line-clamp-2 mb-12`}>{item.description}</p>
                    <div className={`mt-auto flex items-center justify-between pt-10 border-t ${isDarkMode ? 'border-white/5' : 'border-zinc-100'}`}>
                      <p className="text-3xl font-black uppercase italic text-[#D4AF37] tracking-tighter">{formatPrice(item.priceUGX, item.priceUSD)}</p>
                      <button onClick={() => handleBuy(item)} className="w-16 h-16 bg-[#D4AF37] text-black flex items-center justify-center hover:scale-110 active:scale-95 transition-all shadow-xl">
                        <svg className="w-7 h-7" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path strokeLinecap="round" strokeLinejoin="round" strokeWidth={2} d="M12 4v16m8-8H4"/></svg>
                      </button>
                    </div>
                  </div>
                </div>
              ))}
            </div>
          </div>
        );
      case 'artists':
        return (
          <div className="max-w-7xl mx-auto px-4 py-32 page-fade-in">
             <div className={`mb-24 border-b ${isDarkMode ? 'border-white/10' : 'border-zinc-200'} pb-16`}>
                <h1 className="text-7xl md:text-9xl font-black uppercase mb-6 italic text-[#D4AF37] leading-none glow-text">THE ROSTER</h1>
                <p className={`${subTitleColor} font-black uppercase tracking-[0.6em] text-[10px]`}>Active Imperial Personnel Manifest x v4.0</p>
             </div>
             <div className="grid grid-cols-1 md:grid-cols-2 gap-20">
               {ARTISTS.map(artist => (
                 <div key={artist.id} className="group space-y-10">
                   <div className={`aspect-[16/10] overflow-hidden border ${isDarkMode ? 'border-white/5 bg-zinc-900' : 'border-zinc-100 bg-white'} relative cursor-pointer shadow-2xl`} onClick={() => setSelectedArtist(artist)}>
                     <img src={artist.imageUrl} className="w-full h-full object-cover grayscale opacity-40 group-hover:grayscale-0 group-hover:opacity-100 transition-all duration-[1.5s] group-hover:scale-105" alt={artist.name} />
                     <div className={`absolute inset-0 ${isDarkMode ? 'bg-gradient-to-t from-black via-transparent' : 'bg-gradient-to-t from-white/40 via-transparent'}`} />
                     <div className="absolute bottom-10 left-10">
                        <h2 className="text-6xl font-black uppercase italic text-[#D4AF37] leading-none glow-text">{artist.name}</h2>
                        <p className={`${labelColor} font-black uppercase text-[10px] tracking-[0.5em] mt-4`}>{artist.genre}</p>
                     </div>
                   </div>
                   <div className="px-2 max-w-xl">
                     <p className={`${isDarkMode ? 'text-zinc-400' : 'text-zinc-600'} text-lg font-light italic leading-relaxed line-clamp-3 mb-10`}>{artist.bio}</p>
                     <button onClick={() => setSelectedArtist(artist)} className={`text-[10px] font-black uppercase tracking-[0.6em] text-[#D4AF37] border-b-2 ${isDarkMode ? 'border-[#D4AF37]/10' : 'border-[#D4AF37]/30'} pb-3 hover:border-[#D4AF37] transition-all`}>VIEW PERSONNEL FILE</button>
                   </div>
                 </div>
               ))}
             </div>
          </div>
        );
      case 'demo':
        return (
          <div className="max-w-7xl mx-auto px-4 py-32 page-fade-in">
            <div className="grid grid-cols-1 lg:grid-cols-2 gap-32 items-start">
              <div className="sticky top-40 space-y-12">
                <h1 className="text-7xl md:text-9xl font-black uppercase italic mb-8 leading-none text-[#D4AF37] glow-text">RECRUIT</h1>
                <p className={`${labelColor} text-3xl italic font-light leading-relaxed`}>The Empire seeks raw potential and high-frequency vision. Do you possess the sonic standard?</p>
                
                <div 
                  className={`p-10 border transition-all cursor-pointer shadow-2xl ${isLiveActive ? 'border-red-600 bg-red-600/5' : isDarkMode ? 'border-[#D4AF37]/10 bg-zinc-950 hover:bg-zinc-900' : 'border-[#D4AF37]/40 bg-white hover:bg-zinc-50'}`}
                  onClick={startLiveAandR}
                >
                  <div className="flex justify-between items-center mb-10">
                    <span className="text-[#D4AF37] font-black uppercase italic text-2xl tracking-tighter">AI A&R TERMINAL</span>
                    <div className={`w-4 h-4 rounded-full ${isLiveActive ? 'bg-red-600 animate-pulse' : isDarkMode ? 'bg-zinc-800' : 'bg-zinc-200'}`} />
                  </div>
                  <p className="text-zinc-500 text-sm italic mb-10 leading-relaxed">Establish a direct neural link for real-time creative assessment and industry wisdom.</p>
                  <button className={`w-full py-6 font-black uppercase text-[10px] tracking-[0.4em] transition-all shadow-xl ${isLiveActive ? 'bg-red-600 text-white' : 'bg-[#D4AF37] text-black'}`}>
                    {isLiveActive ? 'COMMS LINK ACTIVE' : 'OPEN SECURE LINE'}
                  </button>
                </div>
              </div>

              <div className={`p-12 md:p-16 border border-[#D4AF37]/20 shadow-2xl space-y-12 relative overflow-hidden ${isDarkMode ? 'bg-zinc-950' : 'bg-white'}`}>
                <h2 className="text-4xl font-black uppercase italic text-[#D4AF37]">SUBMISSION ENGINE</h2>
                <form onSubmit={submitDemo} className={`space-y-10 transition-all duration-500 ${isSubmitting ? 'opacity-30 pointer-events-none scale-95' : 'opacity-100'}`}>
                  <div className="space-y-3">
                    <label className={`text-[10px] font-black uppercase ${isDarkMode ? 'text-[#D4AF37]/60' : 'text-[#D4AF37]'} tracking-widest ml-1`}>Imperial Alias</label>
                    <input required type="text" className={`w-full ${isDarkMode ? 'bg-black text-white' : 'bg-zinc-50 text-zinc-900'} border ${isDarkMode ? 'border-white/10' : 'border-zinc-200'} p-6 outline-none focus:border-[#D4AF37] transition-all italic text-xl shadow-inner`} placeholder="STAGE NAME" />
                  </div>
                  <div className="space-y-3">
                    <label className={`text-[10px] font-black uppercase ${isDarkMode ? 'text-[#D4AF37]/60' : 'text-[#D4AF37]'} tracking-widest ml-1`}>Sonic Manifest (Vision / Lyrics)</label>
                    <textarea required value={demoLyrics} onChange={e => setDemoLyrics(e.target.value)} className={`w-full ${isDarkMode ? 'bg-black text-white' : 'bg-zinc-50 text-zinc-900'} border ${isDarkMode ? 'border-white/10' : 'border-zinc-200'} p-6 h-64 outline-none focus:border-[#D4AF37] transition-all resize-none italic text-xl shadow-inner`} placeholder="ENCODE YOUR FREQUENCY..."></textarea>
                  </div>
                  <button type="submit" disabled={isSubmitting || !demoLyrics} className={`w-full py-8 bg-[#D4AF37] text-black font-black uppercase text-xs tracking-[0.4em] hover:bg-[#D4AF37]/80 disabled:opacity-20 shadow-2xl flex items-center justify-center space-x-4 transition-all`}>
                    {isSubmitting && (
                      <div className="w-4 h-4 border-2 border-black/20 border-t-black rounded-full animate-spin" />
                    )}
                    <span>{isSubmitting ? 'ASSESSING FREQUENCIES...' : 'TRANSMIT TO DYNASTY'}</span>
                  </button>
                </form>

                {isSubmitting && (
                  <div className={`absolute inset-0 flex items-center justify-center z-10 ${isDarkMode ? 'bg-black/5' : 'bg-white/5'} backdrop-blur-[1px]`}>
                     <div className="text-center">
                        <div className="w-12 h-12 border-2 border-[#D4AF37]/30 border-t-[#D4AF37] rounded-full animate-spin mx-auto mb-6"></div>
                        <p className="text-[10px] font-black uppercase tracking-[0.6em] text-[#D4AF37] animate-pulse">Scanning Frequencies</p>
                     </div>
                  </div>
                )}

                {demoFeedback && !isSubmitting && (
                  <div className={`mt-16 p-12 border border-[#D4AF37]/40 ${isDarkMode ? 'bg-[#D4AF37]/5' : 'bg-[#D4AF37]/10 shadow-sm'} animate-in slide-in-from-top-10 duration-700`}>
                    <div className={`flex justify-between items-center mb-10 border-b ${isDarkMode ? 'border-[#D4AF37]/10' : 'border-[#D4AF37]/20'} pb-6`}>
                       <h4 className={`${labelColor} font-black uppercase tracking-[0.5em] text-[10px]`}>Imperial Assessment</h4>
                       <span className="text-[#D4AF37] font-black italic text-3xl">{demoFeedback.potential}%</span>
                    </div>
                    <p className={`${isDarkMode ? 'text-white' : 'text-zinc-800'} italic text-3xl leading-snug mb-10`}>"{demoFeedback.feedback}"</p>
                    <div className={`text-[10px] font-black uppercase ${labelColor} tracking-widest`}>
                       SONIC VIBE: {demoFeedback.vibe}
                    </div>
                  </div>
                )}
              </div>
            </div>
          </div>
        );
      case 'login':
        return (
          <div className="max-w-xl mx-auto px-4 py-40 page-fade-in">
            <div className={`p-16 md:p-24 border border-[#D4AF37]/20 shadow-2xl relative text-center ${isDarkMode ? 'bg-zinc-950' : 'bg-white'}`}>
              <h1 className="text-6xl font-black uppercase italic mb-4 text-[#D4AF37] glow-text">AUTHENTICATE</h1>
              <p className={`${subTitleColor} font-black uppercase text-[10px] tracking-[0.8em] mb-20`}>Imperial Member Verification</p>
              
              <form onSubmit={handleLogin} className="space-y-10 text-left">
                <div className="space-y-2">
                  <label className={`text-[10px] font-black uppercase ${isDarkMode ? 'text-[#D4AF37]/60' : 'text-[#D4AF37]'} tracking-widest ml-1`}>Comms ID</label>
                  <input required type="email" value={loginForm.email} onChange={e => setLoginForm({...loginForm, email: e.target.value})} className={`w-full ${isDarkMode ? 'bg-black text-white' : 'bg-zinc-50 text-zinc-900'} border ${isDarkMode ? 'border-white/10' : 'border-zinc-200'} p-6 outline-none focus:border-[#D4AF37] transition-all italic text-xl shadow-inner`} placeholder="IDENTITY@CENTURY.COM" />
                </div>
                <div className="space-y-2">
                  <label className={`text-[10px] font-black uppercase ${isDarkMode ? 'text-[#D4AF37]/60' : 'text-[#D4AF37]'} tracking-widest ml-1`}>Security Cipher</label>
                  <input required type="password" value={loginForm.password} onChange={e => setLoginForm({...loginForm, password: e.target.value})} className={`w-full ${isDarkMode ? 'bg-black text-white' : 'bg-zinc-50 text-zinc-900'} border ${isDarkMode ? 'border-white/10' : 'border-zinc-200'} p-6 outline-none focus:border-[#D4AF37] transition-all italic text-xl shadow-inner`} placeholder="••••••••" />
                </div>
                
                <button 
                  type="button" 
                  onClick={handleRobotCheck} 
                  className={`w-full p-8 border ${isDarkMode ? 'border-white/5 bg-black' : 'border-zinc-200 bg-zinc-50'} flex items-center justify-center space-x-6 transition-all shadow-sm ${isRobotChecked ? 'border-[#D4AF37]/40' : 'hover:border-[#D4AF37]/20'}`}
                >
                  {robotVerifying ? <div className="w-6 h-6 border-2 border-[#D4AF37]/30 border-t-[#D4AF37] rounded-full animate-spin" /> : isRobotChecked ? <svg className={`w-8 h-8 ${isDarkMode ? 'text-[#D4AF37]' : 'text-[#D4AF37]'}`} fill="currentColor" viewBox="0 0 20 20"><path d="M16.707 5.293a1 1 0 010 1.414l-8 8a1 1 0 01-1.414 0l-4-4a1 1 0 011.414-1.414L8 12.586l7.293-7.293a1 1 0 011.414 0z" /></svg> : <div className={`w-8 h-8 border ${isDarkMode ? 'border-white/20' : 'border-zinc-300'}`} />}
                  <span className={`text-[10px] font-black uppercase tracking-[0.4em] ${labelColor}`}>Biological Confirmation</span>
                </button>

                <button type="submit" disabled={!isRobotChecked} className="w-full py-8 bg-[#D4AF37] text-black font-black uppercase text-xs tracking-[0.4em] hover:bg-[#D4AF37]/80 disabled:opacity-20 shadow-2xl transition-all">ESTABLISH CONNECTION</button>
              </form>
            </div>
          </div>
        );
      default:
        return null;
    }
  };

  return (
    <div className={`min-h-screen flex flex-col transition-colors duration-500 ${isDarkMode ? 'bg-[#050505]' : 'bg-[#fdfcf0]'} selection:bg-[#D4AF37] selection:text-black`}>
      <Navbar onNavigate={setActivePage} activePage={activePage} isLoggedIn={user.isLoggedIn} userEmail={user.email} isDarkMode={isDarkMode} onToggleTheme={toggleDarkMode} />
      
      <main className="flex-grow">
        {renderContent()}
      </main>

      <Footer onNavigate={setActivePage} isDarkMode={isDarkMode} />
      
      <MusicPlayer 
        currentTrack={currentTrack} 
        isPlaying={isPlaying} 
        onTogglePlay={() => setIsPlaying(!isPlaying)} 
        onNext={handleNextTrack}
        onPrev={handlePrevTrack}
        isDarkMode={isDarkMode}
      />

      {/* Gallery Modal / Immersive Full-Screen View */}
      {selectedGalleryItem && (
        <div 
          className="fixed inset-0 z-[200] flex items-center justify-center p-4 md:p-12 bg-black/98 backdrop-blur-3xl transition-all duration-500 animate-in fade-in" 
          onClick={() => setSelectedGalleryItem(null)}
        >
          <div className="absolute top-8 right-8 z-20">
            <button 
              onClick={(e) => { e.stopPropagation(); setSelectedGalleryItem(null); }} 
              className="group p-4 text-[#D4AF37] bg-white/5 rounded-full backdrop-blur-md border border-[#D4AF37]/20 hover:bg-[#D4AF37] hover:text-black transition-all"
              aria-label="Close View"
            >
              <svg className="w-10 h-10 group-hover:scale-90 transition-transform" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path strokeLinecap="round" strokeLinejoin="round" strokeWidth={2} d="M6 18L18 6M6 6l12 12"/></svg>
            </button>
          </div>

          <div 
            className="relative max-w-7xl w-full h-full flex flex-col items-center justify-center animate-in zoom-in-95 duration-500" 
            onClick={e => e.stopPropagation()}
          >
            <div className="relative w-full h-[65vh] md:h-[75vh] flex items-center justify-center group overflow-hidden">
              <img 
                src={selectedGalleryItem.imageUrl} 
                className="max-w-full max-h-full object-contain border border-[#D4AF37]/20 shadow-[0_0_100px_rgba(212,175,55,0.1)] transition-all duration-700 hover:scale-[1.01]" 
                alt={selectedGalleryItem.title} 
              />
              <div className="absolute inset-0 pointer-events-none bg-gradient-to-t from-black/60 via-transparent to-transparent opacity-0 group-hover:opacity-100 transition-opacity" />
            </div>
            
            <div className="mt-12 text-center space-y-6 max-w-4xl animate-in slide-in-from-bottom-8 duration-700 delay-150">
               <div>
                  <span className="text-[#D4AF37] font-black uppercase text-xs tracking-[1em] mb-6 block opacity-50">{selectedGalleryItem.category}</span>
                  <h3 className="text-[#D4AF37] font-black uppercase italic text-5xl md:text-7xl leading-tight glow-text drop-shadow-[0_20px_50px_rgba(212,175,55,0.2)]">
                    {selectedGalleryItem.title}
                  </h3>
               </div>
               <div className="w-32 h-px bg-gradient-to-r from-transparent via-[#D4AF37]/40 to-transparent mx-auto" />
               <div className="flex items-center justify-center gap-6">
                  <p className="text-zinc-600 font-mono text-[9px] uppercase tracking-[0.4em]">FILE_ID: {selectedGalleryItem.id.toUpperCase()}</p>
                  <p className="text-zinc-600 font-mono text-[9px] uppercase tracking-[0.4em]">DOMAIN: CENTURY.EMPIRE</p>
               </div>
            </div>
          </div>
        </div>
      )}

      {/* Distribution Checkout Modal */}
      {showDistCheckout && (
        <div className="fixed inset-0 z-[200] flex items-center justify-center p-4 bg-black/90 backdrop-blur-3xl" onClick={() => setShowDistCheckout(false)}>
          <div className={`relative border border-[#D4AF37]/30 max-w-2xl w-full p-12 md:p-16 shadow-2xl animate-in zoom-in-95 ${isDarkMode ? 'bg-zinc-950' : 'bg-white'}`} onClick={e => e.stopPropagation()}>
            <button onClick={() => setShowDistCheckout(false)} className="absolute top-8 right-8 text-[#D4AF37]/40 hover:text-[#D4AF37] transition-colors p-2">
               <svg className="w-8 h-8" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path strokeLinecap="round" strokeLinejoin="round" strokeWidth={2} d="M6 18L18 6M6 6l12 12"/></svg>
            </button>
            
            {distCheckoutStep === 'payment' && (
              <div className="space-y-12">
                <div className="text-center">
                  <h2 className="text-4xl font-black uppercase italic text-[#D4AF37] mb-4">IMPERIAL CHECKOUT</h2>
                  <p className={`${subTitleColor} font-black uppercase tracking-widest text-[10px]`}>ELITE PROTOCOL • 50,000 UGX</p>
                </div>

                <div className="grid grid-cols-2 gap-6">
                  <button 
                    onClick={() => setDistProvider('MTN')}
                    className={`p-8 border flex flex-col items-center justify-center space-y-4 transition-all ${distProvider === 'MTN' ? 'border-[#D4AF37] bg-[#D4AF37]/10' : 'border-white/10 grayscale opacity-40'}`}
                  >
                    <div className="w-12 h-12 bg-yellow-400 flex items-center justify-center rounded-lg font-black text-black">MTN</div>
                    <span className="text-[10px] font-black uppercase tracking-widest">MTN Mobile Money</span>
                  </button>
                  <button 
                    onClick={() => setDistProvider('AIRTEL')}
                    className={`p-8 border flex flex-col items-center justify-center space-y-4 transition-all ${distProvider === 'AIRTEL' ? 'border-[#D4AF37] bg-[#D4AF37]/10' : 'border-white/10 grayscale opacity-40'}`}
                  >
                    <div className="w-12 h-12 bg-red-600 flex items-center justify-center rounded-lg font-black text-white italic">Airtel</div>
                    <span className="text-[10px] font-black uppercase tracking-widest">Airtel Money</span>
                  </button>
                </div>

                <form onSubmit={handlePaymentSubmit} className="space-y-8">
                   <div className="space-y-2">
                      <label className={`text-[10px] font-black uppercase ${labelColor} tracking-widest`}>Recipient Number (UGANDA)</label>
                      <input 
                        required 
                        type="tel" 
                        value={phoneNumber}
                        onChange={e => setPhoneNumber(e.target.value)}
                        className={`w-full ${isDarkMode ? 'bg-black text-white' : 'bg-zinc-50 text-zinc-900'} border ${isDarkMode ? 'border-white/10' : 'border-zinc-200'} p-6 outline-none focus:border-[#D4AF37] transition-all italic text-xl shadow-inner tracking-widest`} 
                        placeholder="07XX XXX XXX" 
                      />
                   </div>
                   <button 
                    type="submit" 
                    disabled={!distProvider || phoneNumber.length < 10}
                    className="w-full py-8 bg-[#D4AF37] text-black font-black uppercase text-xs tracking-[0.4em] shadow-xl hover:scale-[1.02] transition-all disabled:opacity-20"
                   >
                     AUTHORIZE TRANSACTION
                   </button>
                </form>
              </div>
            )}

            {distCheckoutStep === 'processing' && (
              <div className="text-center py-20 space-y-12">
                 <div className="w-24 h-24 border-4 border-[#D4AF37]/20 border-t-[#D4AF37] rounded-full animate-spin mx-auto"></div>
                 <div className="space-y-4">
                    <h3 className="text-3xl font-black italic text-[#D4AF37]">AWAITING NETWORK...</h3>
                    <p className={`${subTitleColor} font-black uppercase tracking-[0.5em] text-[10px]`}>Please check your phone for the Imperial Prompt</p>
                 </div>
              </div>
            )}

            {distCheckoutStep === 'success' && (
              <div className="text-center py-20 space-y-12 animate-in fade-in duration-700">
                 <div className="w-24 h-24 bg-green-600 flex items-center justify-center rounded-full mx-auto shadow-2xl">
                    <svg className="w-12 h-12 text-white" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path strokeLinecap="round" strokeLinejoin="round" strokeWidth={3} d="M5 13l4 4L19 7"/></svg>
                 </div>
                 <div className="space-y-4">
                    <h3 className="text-4xl font-black italic text-[#D4AF37]">BROADCAST SUCCESSFUL</h3>
                    <p className={`${subTitleColor} font-black uppercase tracking-[0.3em] text-[10px]`}>DEPLOYMENT TO 150+ SITES INITIATED</p>
                 </div>
                 <button onClick={() => {setShowDistCheckout(false); setActivePage('music');}} className="px-12 py-5 border border-[#D4AF37] text-[#D4AF37] font-black uppercase text-[10px] tracking-widest hover:bg-[#D4AF37] hover:text-black transition-all">VIEW THE VAULT</button>
              </div>
            )}
          </div>
        </div>
      )}
    </div>
  );
};

export default App;
