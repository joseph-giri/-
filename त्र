import React, { useState, useEffect, useRef, useMemo } from 'react';
import { 
  FileText, 
  Upload, 
  Play, 
  Pause, 
  SkipForward, 
  SkipBack, 
  MessageSquare, 
  BookOpen, 
  Sun, 
  Moon, 
  Search, 
  Highlighter, 
  List, 
  Settings,
  ChevronRight,
  ChevronLeft,
  Volume2,
  X,
  Plus
} from 'lucide-react';

/**
 * TR (त्र) - AI-powered PDF Reader & Natural Audio Listener
 * * Features:
 * - PDF Upload and Rendering (via PDF.js)
 * - AI Chat & Summary (via Gemini 2.5 Flash)
 * - Natural Voice Synth (via Gemini 2.5 Flash TTS)
 * - Highlights & Auto-Notes
 */

const apiKey = ""; // Provided by environment
const modelId = "gemini-2.5-flash-preview-09-2025";
const ttsModelId = "gemini-2.5-flash-preview-tts";

const App = () => {
  const [file, setFile] = useState(null);
  const [pdfData, setPdfData] = useState(null);
  const [currentPage, setCurrentPage] = useState(1);
  const [numPages, setNumPages] = useState(0);
  const [isDarkMode, setIsDarkMode] = useState(false);
  const [activeTab, setActiveTab] = useState('chat'); // chat, notes, chapters
  const [chatMessages, setChatMessages] = useState([]);
  const [inputText, setInputText] = useState("");
  const [isAiLoading, setIsAiLoading] = useState(false);
  const [audioState, setAudioState] = useState({ playing: false, speed: 1.0, progress: 0 });
  const [highlights, setHighlights] = useState([]);
  const [extractedText, setExtractedText] = useState("");
  const [recentFiles, setRecentFiles] = useState([]);
  
  const canvasRef = useRef(null);
  const audioRef = useRef(new Audio());
  const fileInputRef = useRef(null);

  // Initialize PDF.js
  useEffect(() => {
    const script = document.createElement('script');
    script.src = 'https://cdnjs.cloudflare.com/ajax/libs/pdf.js/3.11.174/pdf.min.js';
    script.onload = () => {
      window.pdfjsLib.GlobalWorkerOptions.workerSrc = 'https://cdnjs.cloudflare.com/ajax/libs/pdf.js/3.11.174/pdf.worker.min.js';
    };
    document.head.appendChild(script);
  }, []);

  // Handle PDF Upload
  const handleFileUpload = async (e) => {
    const uploadedFile = e.target.files[0];
    if (!uploadedFile) return;

    setFile(uploadedFile);
    const reader = new FileReader();
    reader.onload = async (event) => {
      const typedarray = new Uint8Array(event.target.result);
      setPdfData(typedarray);
      loadPdf(typedarray, uploadedFile.name);
    };
    reader.readAsArrayBuffer(uploadedFile);
  };

  const loadPdf = async (data, name) => {
    const loadingTask = window.pdfjsLib.getDocument({ data });
    const pdf = await loadingTask.promise;
    setNumPages(pdf.numPages);
    setCurrentPage(1);
    
    // Extract full text for AI context
    let fullText = "";
    for (let i = 1; i <= Math.min(pdf.numPages, 20); i++) { // Limit to 20 pages for MVP speed
      const page = await pdf.getPage(i);
      const textContent = await page.getTextContent();
      fullText += textContent.items.map(item => item.str).join(" ") + "\n";
    }
    setExtractedText(fullText);
    
    // Add to recent
    setRecentFiles(prev => [{ name, date: new Date().toLocaleDateString() }, ...prev.slice(0, 4)]);
    renderPage(1, pdf);
  };

  const renderPage = async (pageNo, pdfInstance = null) => {
    const pdf = pdfInstance || await window.pdfjsLib.getDocument({ data: pdfData }).promise;
    const page = await pdf.getPage(pageNo);
    const scale = 1.5;
    const viewport = page.getViewport({ scale });

    const canvas = canvasRef.current;
    if (!canvas) return;
    const context = canvas.getContext('2d');
    canvas.height = viewport.height;
    canvas.width = viewport.width;

    const renderContext = {
      canvasContext: context,
      viewport: viewport
    };
    await page.render(renderContext).promise;
  };

  useEffect(() => {
    if (pdfData && currentPage) {
      renderPage(currentPage);
    }
  }, [currentPage, pdfData]);

  // AI Functionality: Chat & Summarize
  const callGemini = async (prompt, systemInstruction = "") => {
    setIsAiLoading(true);
    const url = `https://generativelanguage.googleapis.com/v1beta/models/${modelId}:generateContent?key=${apiKey}`;
    
    const payload = {
      contents: [{ parts: [{ text: prompt }] }],
      systemInstruction: systemInstruction ? { parts: [{ text: systemInstruction }] } : undefined
    };

    try {
      let delay = 1000;
      for (let i = 0; i < 5; i++) {
        const response = await fetch(url, {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify(payload)
        });
        if (response.ok) {
          const data = await response.json();
          setIsAiLoading(false);
          return data.candidates?.[0]?.content?.parts?.[0]?.text;
        }
        await new Promise(resolve => setTimeout(resolve, delay));
        delay *= 2;
      }
    } catch (error) {
      console.error("AI Error:", error);
    }
    setIsAiLoading(false);
    return "Error: Could not reach AI assistant.";
  };

  const handleAskQuestion = async () => {
    if (!inputText.trim()) return;
    const userMsg = { role: 'user', text: inputText };
    setChatMessages(prev => [...prev, userMsg]);
    setInputText("");

    const systemPrompt = `You are "त्र" (Tra), a helpful AI assistant reading a PDF. 
    The document text is: ${extractedText.substring(0, 5000)}. 
    Answer questions based ONLY on this text. If not found, say "This information is not found in the document."
    Reference page numbers if possible. Provide answers in the language the user asks (English or Nepali).`;

    const responseText = await callGemini(inputText, systemPrompt);
    setChatMessages(prev => [...prev, { role: 'ai', text: responseText }]);
  };

  const generateSummary = async () => {
    const summaryPrompt = "Please provide a concise summary of this document in bullet points. Highlight the main objective and key takeaways.";
    const systemPrompt = `Analyze this document: ${extractedText.substring(0, 8000)}`;
    const summary = await callGemini(summaryPrompt, systemPrompt);
    setChatMessages(prev => [...prev, { role: 'ai', text: summary }]);
    setActiveTab('chat');
  };

  // Natural TTS using Gemini
  const playNaturalAudio = async (textToSpeak) => {
    if (!textToSpeak) return;
    
    setAudioState(prev => ({ ...prev, playing: true }));
    const url = `https://generativelanguage.googleapis.com/v1beta/models/${ttsModelId}:generateContent?key=${apiKey}`;
    
    const payload = {
      contents: [{ parts: [{ text: `Say in a calm, professional voice: ${textToSpeak.substring(0, 1000)}` }] }],
      generationConfig: {
        responseModalities: ["AUDIO"],
        speechConfig: {
          voiceConfig: { prebuiltVoiceConfig: { voiceName: "Puck" } } // "Puck" or "Charon" for natural male/female
        }
      }
    };

    try {
      const response = await fetch(url, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(payload)
      });
      const result = await response.json();
      const pcmData = result.candidates?.[0]?.content?.parts?.[0]?.inlineData?.data;
      
      if (pcmData) {
        // Convert PCM to WAV for playback
        const audioBlob = pcmToWav(pcmData, 24000); // Typical sample rate for Gemini TTS
        const audioUrl = URL.createObjectURL(audioBlob);
        audioRef.current.src = audioUrl;
        audioRef.current.playbackRate = audioState.speed;
        audioRef.current.play();
        
        audioRef.current.onended = () => {
          setAudioState(prev => ({ ...prev, playing: false }));
        };
      }
    } catch (e) {
      console.error("Audio Error:", e);
      // Fallback to browser TTS if API fails
      const msg = new SpeechSynthesisUtterance(textToSpeak.substring(0, 200));
      msg.rate = audioState.speed;
      window.speechSynthesis.speak(msg);
    }
  };

  // Helper: Simple PCM to WAV conversion for Gemini TTS response
  const pcmToWav = (base64Pcm, sampleRate) => {
    const pcmData = Uint8Array.from(atob(base64Pcm), c => c.charCodeAt(0)).buffer;
    const header = new ArrayBuffer(44);
    const view = new DataView(header);

    const writeString = (offset, string) => {
      for (let i = 0; i < string.length; i++) {
        view.setUint8(offset + i, string.charCodeAt(i));
      }
    };

    writeString(0, 'RIFF');
    view.setUint32(4, 36 + pcmData.byteLength, true);
    writeString(8, 'WAVE');
    writeString(12, 'fmt ');
    view.setUint32(16, 16, true);
    view.setUint16(20, 1, true); // PCM
    view.setUint16(22, 1, true); // Mono
    view.setUint32(24, sampleRate, true);
    view.setUint32(28, sampleRate * 2, true);
    view.setUint16(32, 2, true);
    view.setUint16(34, 16, true);
    writeString(36, 'data');
    view.setUint32(40, pcmData.byteLength, true);

    const blob = new Blob([header, pcmData], { type: 'audio/wav' });
    return blob;
  };

  // UI Components
  const Sidebar = () => (
    <div className={`w-80 border-l flex flex-col ${isDarkMode ? 'bg-slate-900 border-slate-700' : 'bg-slate-50 border-slate-200'}`}>
      <div className="flex border-b">
        <button 
          onClick={() => setActiveTab('chat')}
          className={`flex-1 p-3 text-sm font-medium transition ${activeTab === 'chat' ? (isDarkMode ? 'bg-slate-800 text-blue-400' : 'bg-white text-blue-600') : 'text-slate-500 hover:bg-slate-100'}`}
        >
          <div className="flex items-center justify-center gap-2">
            <MessageSquare size={16} /> AI Chat
          </div>
        </button>
        <button 
          onClick={() => setActiveTab('notes')}
          className={`flex-1 p-3 text-sm font-medium transition ${activeTab === 'notes' ? (isDarkMode ? 'bg-slate-800 text-blue-400' : 'bg-white text-blue-600') : 'text-slate-500 hover:bg-slate-100'}`}
        >
          <div className="flex items-center justify-center gap-2">
            <Highlighter size={16} /> Notes
          </div>
        </button>
      </div>

      <div className="flex-1 overflow-y-auto p-4 space-y-4">
        {activeTab === 'chat' && (
          <>
            {chatMessages.length === 0 && (
              <div className="text-center py-10 space-y-4">
                <div className="bg-blue-100 w-12 h-12 rounded-full flex items-center justify-center mx-auto text-blue-600">
                  <BookOpen size={24} />
                </div>
                <h3 className="font-semibold text-slate-700">Ask your PDF</h3>
                <p className="text-xs text-slate-500 px-6">Get summaries, find key points, or explain complex concepts instantly.</p>
                <button 
                  onClick={generateSummary}
                  className="bg-blue-600 text-white text-sm px-4 py-2 rounded-full hover:bg-blue-700 transition font-medium"
                >
                  Generate Summary
                </button>
              </div>
            )}
            {chatMessages.map((msg, idx) => (
              <div key={idx} className={`max-w-[90%] p-3 rounded-2xl text-sm ${msg.role === 'user' ? 'bg-blue-600 text-white self-end ml-auto rounded-br-none' : (isDarkMode ? 'bg-slate-800 text-slate-200 border border-slate-700' : 'bg-white text-slate-800 border border-slate-200 shadow-sm') + ' rounded-bl-none'}`}>
                {msg.text}
              </div>
            ))}
            {isAiLoading && (
              <div className="flex gap-1 p-2">
                <div className="w-2 h-2 bg-blue-400 rounded-full animate-bounce"></div>
                <div className="w-2 h-2 bg-blue-400 rounded-full animate-bounce delay-100"></div>
                <div className="w-2 h-2 bg-blue-400 rounded-full animate-bounce delay-200"></div>
              </div>
            )}
          </>
        )}
        {activeTab === 'notes' && (
          <div className="space-y-3">
             <div className="bg-amber-50 p-3 rounded-lg border border-amber-200 text-sm text-amber-800">
              Select text in the document to create highlights and auto-notes.
            </div>
            {highlights.map((h, i) => (
              <div key={i} className="p-3 bg-white rounded-lg border shadow-sm">
                <p className="text-sm font-medium text-slate-800 italic">"{h.text}"</p>
                <div className="mt-2 flex justify-between items-center text-[10px] text-slate-500">
                  <span>Page {h.page}</span>
                  <button className="text-red-500">Delete</button>
                </div>
              </div>
            ))}
          </div>
        )}
      </div>

      <div className={`p-4 border-t ${isDarkMode ? 'bg-slate-800 border-slate-700' : 'bg-white border-slate-100'}`}>
        <div className="flex gap-2">
          <input 
            type="text" 
            value={inputText}
            onChange={(e) => setInputText(e.target.value)}
            onKeyDown={(e) => e.key === 'Enter' && handleAskQuestion()}
            placeholder="Ask anything..."
            className={`flex-1 text-sm rounded-lg px-3 py-2 outline-none focus:ring-2 focus:ring-blue-500 transition ${isDarkMode ? 'bg-slate-900 border-slate-600 text-white' : 'bg-slate-100 border-transparent'}`}
          />
          <button 
            disabled={isAiLoading || !inputText.trim()}
            onClick={handleAskQuestion}
            className="bg-blue-600 text-white p-2 rounded-lg hover:bg-blue-700 disabled:opacity-50"
          >
            <ChevronRight size={18} />
          </button>
        </div>
      </div>
    </div>
  );

  return (
    <div className={`flex flex-col h-screen font-sans transition-colors duration-300 ${isDarkMode ? 'bg-slate-950 text-slate-100' : 'bg-white text-slate-900'}`}>
      {/* Top Header */}
      <header className={`px-6 h-16 flex items-center justify-between border-b ${isDarkMode ? 'border-slate-800 bg-slate-950' : 'border-slate-100 bg-white shadow-sm'} z-20`}>
        <div className="flex items-center gap-3">
          <div className="w-10 h-10 bg-blue-600 rounded-xl flex items-center justify-center text-white font-bold text-2xl">
            त्र
          </div>
          <div>
            <h1 className="text-lg font-bold tracking-tight">Tra <span className="text-blue-500">AI</span></h1>
            <p className="text-[10px] text-slate-400 font-medium uppercase tracking-widest">Read • Listen • Learn</p>
          </div>
        </div>

        <div className="flex items-center gap-4">
          <button 
            onClick={() => setIsDarkMode(!isDarkMode)}
            className={`p-2 rounded-full hover:bg-slate-100 transition ${isDarkMode ? 'hover:bg-slate-800' : ''}`}
          >
            {isDarkMode ? <Sun size={20} className="text-amber-400" /> : <Moon size={20} className="text-slate-600" />}
          </button>
          
          <button 
            onClick={() => fileInputRef.current?.click()}
            className="flex items-center gap-2 bg-blue-600 hover:bg-blue-700 text-white px-4 py-2 rounded-full text-sm font-medium transition shadow-md shadow-blue-500/20"
          >
            <Plus size={18} />
            <span className="hidden sm:inline">New Document</span>
          </button>
          <input 
            type="file" 
            ref={fileInputRef} 
            onChange={handleFileUpload} 
            accept="application/pdf" 
            className="hidden" 
          />
        </div>
      </header>

      {/* Main Content Area */}
      <main className="flex-1 flex overflow-hidden">
        {!file ? (
          <div className="flex-1 flex items-center justify-center p-8">
            <div className="max-w-4xl w-full grid grid-cols-1 md:grid-cols-2 gap-12 items-center">
              <div className="space-y-6">
                <div className="space-y-2">
                  <h2 className="text-4xl font-extrabold leading-tight">
                    Transform your <span className="text-blue-600 underline underline-offset-4">reading</span> into listening.
                  </h2>
                  <p className="text-slate-500 text-lg">
                    Upload your textbooks, PDFs, or research papers and experience them in natural human-like voices. Get AI summaries instantly.
                  </p>
                </div>
                
                <div 
                  onClick={() => fileInputRef.current?.click()}
                  className={`border-2 border-dashed rounded-3xl p-10 text-center cursor-pointer transition hover:border-blue-400 hover:bg-blue-50/50 group ${isDarkMode ? 'border-slate-800' : 'border-slate-200'}`}
                >
                  <div className="bg-blue-100 w-16 h-16 rounded-full flex items-center justify-center mx-auto mb-4 group-hover:scale-110 transition">
                    <Upload className="text-blue-600" size={28} />
                  </div>
                  <h3 className="font-semibold text-lg">Click to Upload PDF</h3>
                  <p className="text-sm text-slate-400 mt-1">Maximum file size: 25MB</p>
                </div>
                
                <div className="flex items-center gap-4 text-sm text-slate-400 font-medium">
                  <div className="flex -space-x-2">
                    {[1,2,3].map(i => <div key={i} className={`w-8 h-8 rounded-full border-2 ${isDarkMode ? 'border-slate-900' : 'border-white'} bg-slate-300`}></div>)}
                  </div>
                  <span>Trusted by 5,000+ students in Nepal</span>
                </div>
              </div>

              <div className={`rounded-3xl p-6 ${isDarkMode ? 'bg-slate-900' : 'bg-slate-50 border border-slate-100'}`}>
                <h3 className="font-bold mb-4 flex items-center gap-2">
                   <List size={18} className="text-blue-500" /> Recent Documents
                </h3>
                {recentFiles.length > 0 ? (
                  <div className="space-y-3">
                    {recentFiles.map((rf, i) => (
                      <div key={i} className={`p-4 rounded-xl flex items-center justify-between group cursor-pointer transition ${isDarkMode ? 'bg-slate-800 hover:bg-slate-700' : 'bg-white hover:shadow-md'}`}>
                        <div className="flex items-center gap-3">
                          <FileText size={20} className="text-blue-500" />
                          <div>
                            <p className="font-semibold text-sm line-clamp-1">{rf.name}</p>
                            <p className="text-[10px] text-slate-400 uppercase tracking-wider">{rf.date}</p>
                          </div>
                        </div>
                        <ChevronRight size={16} className="text-slate-300 opacity-0 group-hover:opacity-100 transition" />
                      </div>
                    ))}
                  </div>
                ) : (
                  <div className="py-20 text-center space-y-2 opacity-60">
                    <FileText size={48} className="mx-auto text-slate-300" />
                    <p className="text-sm">No documents found.</p>
                  </div>
                )}
              </div>
            </div>
          </div>
        ) : (
          <div className="flex-1 flex">
            {/* PDF Viewer */}
            <div className={`flex-1 flex flex-col items-center overflow-y-auto p-4 sm:p-8 ${isDarkMode ? 'bg-slate-900' : 'bg-slate-100'}`}>
              <div className={`bg-white shadow-2xl rounded-lg mb-20 relative select-text`}>
                <canvas 
                  ref={canvasRef} 
                  className="max-w-full h-auto rounded-lg"
                  onMouseUp={() => {
                    const selected = window.getSelection().toString();
                    if (selected) {
                      setHighlights(prev => [...prev, { text: selected, page: currentPage }]);
                    }
                  }}
                />
              </div>

              {/* Floating Page Controls */}
              <div className={`fixed bottom-24 left-1/2 -translate-x-1/2 flex items-center gap-3 p-2 rounded-full shadow-lg border backdrop-blur-md ${isDarkMode ? 'bg-slate-800/90 border-slate-700' : 'bg-white/90 border-slate-200'} transition-all hover:scale-105`}>
                <button 
                  disabled={currentPage === 1}
                  onClick={() => setCurrentPage(p => Math.max(1, p - 1))}
                  className="p-2 rounded-full hover:bg-slate-200/50 disabled:opacity-30"
                >
                  <ChevronLeft size={20} />
                </button>
                <span className="text-sm font-bold px-2 whitespace-nowrap">
                  {currentPage} <span className="text-slate-400 font-normal">/ {numPages}</span>
                </span>
                <button 
                  disabled={currentPage === numPages}
                  onClick={() => setCurrentPage(p => Math.min(numPages, p + 1))}
                  className="p-2 rounded-full hover:bg-slate-200/50 disabled:opacity-30"
                >
                  <ChevronRight size={20} />
                </button>
                <div className="w-px h-6 bg-slate-300/50 mx-1"></div>
                <button 
                   onClick={() => playNaturalAudio(extractedText.substring(0, 500))}
                   className="flex items-center gap-2 bg-blue-100 text-blue-600 px-3 py-1.5 rounded-full text-xs font-bold hover:bg-blue-200 transition"
                >
                  <Volume2 size={14} /> Listen Page
                </button>
              </div>
            </div>

            {/* Sidebar (AI Chat & Notes) */}
            <Sidebar />
          </div>
        )}
      </main>

      {/* Media Player Controls (Visible when doc is open) */}
      {file && (
        <footer className={`h-20 border-t flex items-center px-6 fixed bottom-0 left-0 right-0 z-30 ${isDarkMode ? 'bg-slate-950 border-slate-800' : 'bg-white border-slate-200'} shadow-[0_-4px_20px_-5px_rgba(0,0,0,0.1)]`}>
          <div className="flex-1 flex items-center gap-4">
            <div className="hidden sm:block">
              <p className="font-bold text-sm line-clamp-1">{file.name}</p>
              <p className="text-[10px] text-blue-500 font-bold uppercase tracking-widest">Page {currentPage} of {numPages}</p>
            </div>
          </div>

          <div className="flex flex-col items-center gap-1 flex-1">
            <div className="flex items-center gap-6">
              <button className="text-slate-400 hover:text-slate-600"><SkipBack size={20} /></button>
              <button 
                onClick={() => {
                   if (audioRef.current.paused) audioRef.current.play();
                   else audioRef.current.pause();
                   setAudioState(prev => ({ ...prev, playing: !prev.playing }));
                }}
                className="w-12 h-12 bg-blue-600 rounded-full flex items-center justify-center text-white hover:scale-105 active:scale-95 transition shadow-lg shadow-blue-500/30"
              >
                {audioState.playing ? <Pause size={24} fill="currentColor" /> : <Play size={24} fill="currentColor" className="ml-1" />}
              </button>
              <button className="text-slate-400 hover:text-slate-600"><SkipForward size={20} /></button>
            </div>
            <div className="w-full max-w-xs h-1 bg-slate-100 rounded-full overflow-hidden mt-2">
               <div className="h-full bg-blue-600 w-1/3 rounded-full"></div>
            </div>
          </div>

          <div className="flex-1 flex items-center justify-end gap-4">
            <div className="flex items-center gap-2 bg-slate-100 dark:bg-slate-800 px-3 py-1 rounded-full text-xs font-bold">
              <span className="text-slate-400">Speed:</span>
              <button 
                onClick={() => {
                  const newSpeed = audioState.speed >= 1.5 ? 0.75 : audioState.speed + 0.25;
                  setAudioState(prev => ({ ...prev, speed: newSpeed }));
                  audioRef.current.playbackRate = newSpeed;
                }}
                className="hover:text-blue-500 transition"
              >
                {audioState.speed}x
              </button>
            </div>
            <button 
              onClick={() => setFile(null)}
              className="p-2 text-slate-400 hover:text-red-500 transition"
            >
              <X size={20} />
            </button>
          </div>
        </footer>
      )}
      
      {/* Toast Notification (Simple) */}
      <style>{`
        @import url('https://fonts.googleapis.com/css2?family=Noto+Sans+Devanagari:wght@400;700&display=swap');
        body { font-family: 'Inter', 'Noto Sans Devanagari', sans-serif; }
        .line-clamp-1 { display: -webkit-box; -webkit-line-clamp: 1; -webkit-box-orient: vertical; overflow: hidden; }
        canvas { user-select: text; }
      `}</style>
    </div>
  );
};

export default App;
