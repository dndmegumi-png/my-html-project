import React, { useState, useEffect, useMemo, useRef } from 'react';
import { initializeApp } from 'firebase/app';
import { getAuth, signInAnonymously, onAuthStateChanged } from 'firebase/auth';
import { getFirestore, collection, doc, setDoc, onSnapshot, query, addDoc, updateDoc, deleteDoc, Timestamp } from 'firebase/firestore';
import { 
  Search, Plus, UserPlus, Package, CheckCircle, Clock, Trash2, Shield, 
  X, BarChart3, Users, Tag, LayoutGrid, Sparkles, MessageSquare, Send, Bot,
  TrendingUp, Wallet, Banknote, ArrowUpRight, ArrowDownRight
} from 'lucide-react';

// --- Firebase Configuration ---
const firebaseConfig = JSON.parse(__firebase_config);
const app = initializeApp(firebaseConfig);
const auth = getAuth(app);
const db = getFirestore(app);
const appId = typeof __app_id !== 'undefined' ? __app_id : 'tdgc-ultimate-tracker-v3';
const apiKey = ""; // API key is provided by the environment at runtime

const App = () => {
  const [user, setUser] = useState(null);
  const [resellers, setResellers] = useState([]);
  const [orders, setOrders] = useState([]);
  const [categories, setCategories] = useState([]);
  const [activeTab, setActiveTab] = useState('orders');
  const [searchTerm, setSearchTerm] = useState('');

  // AI & Modal States
  const [aiLoading, setAiLoading] = useState(false);
  const [aiModalContent, setAiModalContent] = useState(null);
  const [showChat, setShowChat] = useState(false);
  const [chatInput, setChatInput] = useState('');
  const [chatMessages, setChatMessages] = useState([
    { role: 'assistant', text: 'Hello! I am your ✨ TDGC Business Assistant. Ask me anything about your capital, ROI, or profits!' }
  ]);
  const chatEndRef = useRef(null);

  // Form States
  const [newReseller, setNewReseller] = useState({ name: '', initials: '' });
  const [newCategory, setNewCategory] = useState('');
  const [newOrder, setNewOrder] = useState({ 
    customerType: 'Reseller', 
    resellerId: '', 
    directBuyerName: '', 
    amount: '', 
    cost: '', // Capital Field
    credits: '', 
    category: '',
    date: new Date().toISOString().split('T')[0]
  });

  // 1. Auth Logic (MANDATORY RULE 3)
  useEffect(() => {
    const initAuth = async () => {
      try {
        if (typeof __initial_auth_token !== 'undefined' && __initial_auth_token) {
          await signInAnonymously(auth);
        } else {
          await signInAnonymously(auth);
        }
      } catch (err) {
        console.error("Auth error:", err);
      }
    };
    initAuth();
    const unsubscribe = onAuthStateChanged(auth, setUser);
    return () => unsubscribe();
  }, []);

  // 2. Data Fetching (MANDATORY RULE 1 & 2)
  useEffect(() => {
    if (!user) return;

    const qResellers = query(collection(db, 'artifacts', appId, 'public', 'data', 'resellers'));
    const unsubResellers = onSnapshot(qResellers, (snapshot) => {
      setResellers(snapshot.docs.map(doc => ({ id: doc.id, ...doc.data() })));
    }, (err) => console.error(err));

    const qCategories = query(collection(db, 'artifacts', appId, 'public', 'data', 'categories'));
    const unsubCategories = onSnapshot(qCategories, (snapshot) => {
      const data = snapshot.docs.map(doc => ({ id: doc.id, ...doc.data() }));
      setCategories(data.length > 0 ? data : [{ id: 'default', name: 'General' }]);
    }, (err) => console.error(err));

    const qOrders = query(collection(db, 'artifacts', appId, 'public', 'data', 'orders'));
    const unsubOrders = onSnapshot(qOrders, (snapshot) => {
      const data = snapshot.docs.map(doc => ({ id: doc.id, ...doc.data() }));
      setOrders(data.sort((a, b) => new Date(b.date) - new Date(a.date)));
    }, (err) => console.error(err));

    return () => {
      unsubResellers();
      unsubCategories();
      unsubOrders();
    };
  }, [user]);

  useEffect(() => {
    if (categories.length > 0 && !newOrder.category) {
      setNewOrder(prev => ({ ...prev, category: categories[0].name }));
    }
  }, [categories]);

  useEffect(() => {
    chatEndRef.current?.scrollIntoView({ behavior: 'smooth' });
  }, [chatMessages]);

  // --- Gemini API Logic ---
  const callGemini = async (prompt, systemInstruction = "You are an AI assistant for TDGC, a game credits business.", retryCount = 0) => {
    try {
      const response = await fetch(`https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash-preview-09-2025:generateContent?key=${apiKey}`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          contents: [{ parts: [{ text: prompt }] }],
          systemInstruction: { parts: [{ text: systemInstruction }] }
        })
      });
      if (!response.ok) throw new Error('Gemini API Error');
      const result = await response.json();
      return result.candidates?.[0]?.content?.parts?.[0]?.text || "I'm sorry, I couldn't generate a response.";
    } catch (err) {
      if (retryCount < 5) {
        await new Promise(r => setTimeout(r, Math.pow(2, retryCount) * 1000));
        return callGemini(prompt, systemInstruction, retryCount + 1);
      }
      throw err;
    }
  };

  const generateAIInsights = async () => {
    setAiLoading(true);
    const categoryPerformance = categories.map(c => ({
      name: c.name,
      revenue: orders.filter(o => o.category === c.name).reduce((acc, curr) => acc + Number(curr.amount), 0),
      profit: orders.filter(o => o.category === c.name).reduce((acc, curr) => acc + (Number(curr.amount) - Number(curr.cost || 0)), 0)
    })).sort((a, b) => b.profit - a.profit);

    const prompt = `Analyze this sales data:
    Total Revenue: ₱${stats.total}
    Total Capital/Cost: ₱${stats.totalCost}
    Net Profit: ₱${stats.totalProfit}
    Categories: ${JSON.stringify(categoryPerformance)}

    Give me a 3-point breakdown on how to grow this capital. Be specific about which games give the best ROI.`;

    try {
      const insight = await callGemini(prompt, "You are a Financial and Business Intelligence expert for game credit trading.");
      setAiModalContent({ title: "✨ TDGC Profit Strategy", text: insight });
    } catch (err) {
      setAiModalContent({ title: "Error", text: "AI is currently offline." });
    } finally {
      setAiLoading(false);
    }
  };

  const handleChatSubmit = async (e) => {
    e.preventDefault();
    if (!chatInput.trim()) return;

    const userMsg = chatInput;
    setChatInput('');
    setChatMessages(prev => [...prev, { role: 'user', text: userMsg }]);
    setAiLoading(true);

    const businessContext = `Business State: Revenue ₱${stats.total}, Capital ₱${stats.totalCost}, Profit ₱${stats.totalProfit}. Orders: ${orders.length}.`;
    
    try {
      const response = await callGemini(`${businessContext}\nUser question: ${userMsg}`, "You are the TDGC Assistant. Be brief and focus on financial growth.");
      setChatMessages(prev => [...prev, { role: 'assistant', text: response }]);
    } catch (err) {
      setChatMessages(prev => [...prev, { role: 'assistant', text: "Connection error." }]);
    } finally {
      setAiLoading(false);
    }
  };

  // --- Database Actions ---
  const handleAddCategory = async (e) => {
    e.preventDefault();
    if (!newCategory || !user) return;
    try {
      await addDoc(collection(db, 'artifacts', appId, 'public', 'data', 'categories'), { name: newCategory });
      setNewCategory('');
    } catch (err) { console.error(err); }
  };

  const deleteCategory = async (id) => {
    if (!user) return;
    await deleteDoc(doc(db, 'artifacts', appId, 'public', 'data', 'categories', id));
  };

  const handleAddReseller = async (e) => {
    e.preventDefault();
    if (!newReseller.name || !newReseller.initials || !user) return;
    try {
      await addDoc(collection(db, 'artifacts', appId, 'public', 'data', 'resellers'), {
        name: newReseller.name,
        initials: newReseller.initials.toUpperCase(),
        createdAt: Timestamp.now()
      });
      setNewReseller({ name: '', initials: '' });
    } catch (err) { console.error(err); }
  };

  const handleAddOrder = async (e) => {
    e.preventDefault();
    if (!user) return;
    
    let customerName = "";
    let initials = "END";
    
    if (newOrder.customerType === 'Reseller') {
      const reseller = resellers.find(r => r.id === newOrder.resellerId);
      if (!reseller) return;
      customerName = reseller.name;
      initials = reseller.initials;
    } else {
      if (!newOrder.directBuyerName) return;
      customerName = newOrder.directBuyerName;
      initials = "DIR";
    }

    const dateStr = newOrder.date.replace(/-/g, '').slice(2, 6);
    const sequence = (orders.length + 1).toString().padStart(4, '0');
    const invoiceCode = `TDGC-${dateStr}-${initials}-${sequence}`;

    try {
      await addDoc(collection(db, 'artifacts', appId, 'public', 'data', 'orders'), {
        ...newOrder,
        customerName,
        invoiceCode,
        createdAt: Timestamp.now(),
        amount: Number(newOrder.amount),
        cost: Number(newOrder.cost || 0)
      });
      setNewOrder({ ...newOrder, amount: '', cost: '', credits: '', directBuyerName: '' });
    } catch (err) { console.error(err); }
  };

  const deleteOrder = async (id) => {
    if (!user) return;
    await deleteDoc(doc(db, 'artifacts', appId, 'public', 'data', 'orders', id));
  };

  const stats = useMemo(() => {
    const total = orders.reduce((a, b) => a + Number(b.amount), 0);
    const totalCost = orders.reduce((a, b) => a + Number(b.cost || 0), 0);
    const totalProfit = total - totalCost;
    return { total, totalCost, totalProfit };
  }, [orders]);

  const filteredOrders = useMemo(() => {
    return orders.filter(o => 
      o.invoiceCode.toLowerCase().includes(searchTerm.toLowerCase()) || 
      o.customerName?.toLowerCase().includes(searchTerm.toLowerCase()) ||
      o.category?.toLowerCase().includes(searchTerm.toLowerCase())
    );
  }, [orders, searchTerm]);

  if (!user) return <div className="h-screen flex flex-col items-center justify-center bg-slate-50 font-black text-red-700 gap-4">
    <div className="animate-spin border-4 border-red-700 border-t-transparent h-12 w-12 rounded-full"></div>
    SYNCING TDGC CLOUD...
  </div>;

  return (
    <div className="min-h-screen bg-slate-50 text-slate-900 font-sans pb-32">
      {/* AI Modal */}
      {aiModalContent && (
        <div className="fixed inset-0 z-[100] flex items-center justify-center bg-slate-900/60 backdrop-blur-sm p-4">
          <div className="bg-white rounded-3xl w-full max-w-xl shadow-2xl overflow-hidden">
            <div className="p-8">
              <div className="flex justify-between items-center mb-6">
                <h3 className="text-xl font-black flex items-center gap-2">
                  <Sparkles className="text-red-700 w-5 h-5" /> {aiModalContent.title}
                </h3>
                <button onClick={() => setAiModalContent(null)} className="text-slate-400 hover:text-slate-600 transition"><X /></button>
              </div>
              <div className="bg-slate-50 p-6 rounded-2xl border border-slate-100 text-slate-700 leading-relaxed max-h-[60vh] overflow-y-auto font-medium">
                {aiModalContent.text}
              </div>
              <button onClick={() => setAiModalContent(null)} className="w-full mt-6 bg-slate-900 text-white font-black py-4 rounded-2xl hover:bg-black transition">Done</button>
            </div>
          </div>
        </div>
      )}

      {/* Floating Chat Bot */}
      <div className="fixed bottom-24 right-6 z-50 flex flex-col items-end gap-4">
        {showChat && (
          <div className="bg-white w-[350px] h-[500px] rounded-3xl shadow-2xl border border-slate-200 flex flex-col overflow-hidden animate-in slide-in-from-bottom-10 duration-300">
            <div className="bg-slate-900 p-4 text-white flex justify-between items-center">
              <div className="flex items-center gap-2">
                <Bot className="w-5 h-5 text-red-500" />
                <span className="font-black text-xs uppercase tracking-widest">Assistant</span>
              </div>
              <button onClick={() => setShowChat(false)}><X className="w-4 h-4" /></button>
            </div>
            <div className="flex-1 overflow-y-auto p-4 space-y-4 bg-slate-50">
              {chatMessages.map((m, i) => (
                <div key={i} className={`flex ${m.role === 'user' ? 'justify-end' : 'justify-start'}`}>
                  <div className={`max-w-[80%] p-3 rounded-2xl text-xs font-bold leading-relaxed ${m.role === 'user' ? 'bg-red-700 text-white rounded-br-none' : 'bg-white border border-slate-200 text-slate-700 rounded-bl-none shadow-sm'}`}>
                    {m.text}
                  </div>
                </div>
              ))}
              {aiLoading && <div className="flex justify-start animate-pulse"><div className="bg-white p-3 rounded-2xl border border-slate-200 text-[10px] font-black uppercase text-slate-400">Thinking...</div></div>}
              <div ref={chatEndRef} />
            </div>
            <form onSubmit={handleChatSubmit} className="p-3 bg-white border-t border-slate-100 flex gap-2">
              <input 
                type="text" 
                value={chatInput}
                onChange={e => setChatInput(e.target.value)}
                placeholder="Ask about capital..."
                className="flex-1 bg-slate-50 p-3 rounded-xl text-xs font-bold outline-none"
              />
              <button type="submit" className="bg-red-700 text-white p-3 rounded-xl"><Send className="w-4 h-4" /></button>
            </form>
          </div>
        )}
        <button onClick={() => setShowChat(!showChat)} className="bg-red-700 text-white w-14 h-14 rounded-full shadow-xl flex items-center justify-center hover:scale-110 transition"><MessageSquare /></button>
      </div>

      <header className="bg-white border-b border-slate-200 sticky top-0 z-20 shadow-sm">
        <div className="max-w-7xl mx-auto px-4 py-4 flex flex-col md:flex-row justify-between items-center gap-4">
          <div className="flex items-center gap-4">
            <div className="bg-red-700 p-2 rounded-xl shadow-lg text-white font-black italic">TDGC</div>
            <div>
              <h1 className="text-lg font-black tracking-tighter leading-none uppercase">Advanced Tracker</h1>
              <p className="text-[10px] font-bold text-slate-400 uppercase tracking-widest mt-1">Capital & Profit Control</p>
            </div>
          </div>
          <div className="flex bg-slate-100 p-1 rounded-2xl border border-slate-200 overflow-x-auto no-scrollbar">
            {[
              { id: 'orders', icon: Package, label: 'Sales' },
              { id: 'resellers', icon: Users, label: 'Partners' },
              { id: 'categories', icon: LayoutGrid, label: 'Games' },
              { id: 'analytics', icon: BarChart3, label: 'Financials' }
            ].map(tab => (
              <button 
                key={tab.id}
                onClick={() => setActiveTab(tab.id)}
                className={`flex items-center gap-2 px-6 py-2.5 rounded-xl text-xs font-black transition-all ${activeTab === tab.id ? 'bg-white text-red-700 shadow-sm' : 'text-slate-500 hover:text-slate-800 whitespace-nowrap'}`}
              >
                <tab.icon className="w-4 h-4" /> {tab.label}
              </button>
            ))}
          </div>
        </div>
      </header>

      <main className="max-w-7xl mx-auto px-4 py-8">
        {activeTab === 'orders' ? (
          <div className="space-y-8 animate-in fade-in duration-300">
            <div className="bg-white p-8 rounded-3xl border border-slate-200 shadow-sm">
              <h2 className="text-[10px] font-black text-slate-400 uppercase tracking-widest mb-6">New Transaction</h2>
              <form onSubmit={handleAddOrder} className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-4">
                <div className="flex flex-col gap-2">
                  <label className="text-[9px] font-black uppercase text-slate-400">Customer</label>
                  {newOrder.customerType === 'Reseller' ? (
                    <select 
                      value={newOrder.resellerId} 
                      onChange={e => setNewOrder({...newOrder, resellerId: e.target.value})}
                      className="p-4 bg-slate-50 border border-slate-200 rounded-2xl font-bold outline-none"
                      required
                    >
                      <option value="">Select Reseller</option>
                      {resellers.map(r => <option key={r.id} value={r.id}>{r.name}</option>)}
                    </select>
                  ) : (
                    <input type="text" placeholder="Buyer Name" value={newOrder.directBuyerName} onChange={e => setNewOrder({...newOrder, directBuyerName: e.target.value})} className="p-4 bg-slate-50 border border-slate-200 rounded-2xl font-bold outline-none" required />
                  )}
                  <button type="button" onClick={() => setNewOrder({...newOrder, customerType: newOrder.customerType === 'Reseller' ? 'Direct' : 'Reseller'})} className="text-[9px] font-black text-red-700 text-left uppercase">Switch to {newOrder.customerType === 'Reseller' ? 'End Buyer' : 'Reseller'}</button>
                </div>

                <div className="flex flex-col gap-2">
                  <label className="text-[9px] font-black uppercase text-slate-400">Game & Product</label>
                  <div className="flex gap-2">
                    <select value={newOrder.category} onChange={e => setNewOrder({...newOrder, category: e.target.value})} className="flex-1 p-4 bg-slate-50 border border-slate-200 rounded-2xl font-bold outline-none">
                      {categories.map(c => <option key={c.id} value={c.name}>{c.name}</option>)}
                    </select>
                    <input type="text" placeholder="Credits" value={newOrder.credits} onChange={e => setNewOrder({...newOrder, credits: e.target.value})} className="flex-1 p-4 bg-slate-50 border border-slate-200 rounded-2xl font-bold outline-none" required />
                  </div>
                </div>

                <div className="grid grid-cols-2 gap-4">
                  <div className="flex flex-col gap-2">
                    <label className="text-[9px] font-black uppercase text-slate-400">Capital (Cost)</label>
                    <input type="number" placeholder="₱" value={newOrder.cost} onChange={e => setNewOrder({...newOrder, cost: e.target.value})} className="p-4 bg-slate-50 border border-red-200 rounded-2xl font-bold outline-none text-red-700" required />
                  </div>
                  <div className="flex flex-col gap-2">
                    <label className="text-[9px] font-black uppercase text-slate-400">Selling Price</label>
                    <input type="number" placeholder="₱" value={newOrder.amount} onChange={e => setNewOrder({...newOrder, amount: e.target.value})} className="p-4 bg-slate-50 border border-green-200 rounded-2xl font-bold outline-none text-green-700" required />
                  </div>
                </div>

                <div className="lg:col-span-3">
                  <button type="submit" className="w-full bg-red-700 text-white font-black py-4 rounded-2xl hover:bg-red-800 transition shadow-xl shadow-red-700/20 flex items-center justify-center gap-2 uppercase text-xs tracking-widest">
                    <CheckCircle className="w-4 h-4" /> Save Record
                  </button>
                </div>
              </form>
            </div>

            <div className="space-y-4">
              <div className="relative">
                <Search className="absolute left-5 top-1/2 -translate-y-1/2 text-slate-400 w-4 h-4" />
                <input type="text" placeholder="Search orders..." value={searchTerm} onChange={e => setSearchTerm(e.target.value)} className="w-full pl-14 pr-4 py-5 bg-white border border-slate-200 rounded-3xl shadow-sm outline-none font-bold" />
              </div>

              <div className="bg-white rounded-[2rem] border border-slate-200 overflow-hidden shadow-sm">
                <div className="overflow-x-auto">
                  <table className="w-full text-left">
                    <thead className="bg-slate-50 text-[10px] font-black uppercase text-slate-400 border-b">
                      <tr>
                        <th className="px-8 py-5">Invoice / Date</th>
                        <th className="px-8 py-5">Customer / Game</th>
                        <th className="px-8 py-5 text-right">Capital</th>
                        <th className="px-8 py-5 text-right">Selling Price</th>
                        <th className="px-8 py-5 text-right">Net Profit</th>
                        <th className="px-8 py-5 text-right">Action</th>
                      </tr>
                    </thead>
                    <tbody className="divide-y">
                      {filteredOrders.length === 0 ? (
                        <tr><td colSpan="6" className="px-8 py-20 text-center font-black text-slate-300 uppercase text-xs">No records found</td></tr>
                      ) : (
                        filteredOrders.map(o => {
                          const profit = Number(o.amount) - Number(o.cost || 0);
                          return (
                            <tr key={o.id} className="hover:bg-slate-50 transition-colors group">
                              <td className="px-8 py-6">
                                <p className="font-mono font-black text-slate-900 text-xs">{o.invoiceCode}</p>
                                <p className="text-[10px] font-bold text-slate-400 mt-1">{o.date}</p>
                              </td>
                              <td className="px-8 py-6">
                                <p className="font-black text-slate-800">{o.customerName}</p>
                                <p className="text-[9px] font-bold text-slate-400 uppercase">{o.category} - {o.credits}</p>
                              </td>
                              <td className="px-8 py-6 text-right font-bold text-slate-400 text-sm">₱{Number(o.cost || 0).toLocaleString()}</td>
                              <td className="px-8 py-6 text-right font-black text-slate-900">₱{Number(o.amount).toLocaleString()}</td>
                              <td className="px-8 py-6 text-right">
                                <div className={`inline-flex items-center gap-1 font-black text-xs px-3 py-1 rounded-full ${profit >= 0 ? 'bg-green-50 text-green-700' : 'bg-red-50 text-red-700'}`}>
                                  {profit >= 0 ? <ArrowUpRight className="w-3 h-3" /> : <ArrowDownRight className="w-3 h-3" />}
                                  ₱{profit.toLocaleString()}
                                </div>
                              </td>
                              <td className="px-8 py-6 text-right">
                                <button onClick={() => deleteOrder(o.id)} className="text-slate-200 hover:text-red-600 transition p-2"><Trash2 className="w-4 h-4" /></button>
                              </td>
                            </tr>
                          );
                        })
                      )}
                    </tbody>
                  </table>
                </div>
              </div>
            </div>
          </div>
        ) : activeTab === 'analytics' ? (
          <div className="space-y-8 animate-in fade-in duration-500">
             <div className="grid grid-cols-1 md:grid-cols-3 gap-6">
                <div className="bg-white p-8 rounded-3xl border border-slate-200 shadow-sm">
                  <div className="flex items-center gap-3 mb-4">
                    <div className="p-2 bg-blue-50 text-blue-600 rounded-xl"><Wallet className="w-5 h-5" /></div>
                    <p className="text-[10px] font-black text-slate-400 uppercase">Gross Revenue</p>
                  </div>
                  <p className="text-3xl font-black text-slate-900">₱{stats.total.toLocaleString()}</p>
                </div>
                
                <div className="bg-white p-8 rounded-3xl border border-slate-200 shadow-sm">
                  <div className="flex items-center gap-3 mb-4">
                    <div className="p-2 bg-slate-50 text-slate-600 rounded-xl"><Banknote className="w-5 h-5" /></div>
                    <p className="text-[10px] font-black text-slate-400 uppercase">Capital Invested</p>
                  </div>
                  <p className="text-3xl font-black text-slate-600">₱{stats.totalCost.toLocaleString()}</p>
                </div>

                <div className="bg-slate-900 p-8 rounded-3xl text-white shadow-xl">
                  <div className="flex items-center gap-3 mb-4">
                    <div className="p-2 bg-green-500/20 text-green-400 rounded-xl"><TrendingUp className="w-5 h-5" /></div>
                    <p className="text-[10px] font-black text-slate-300 uppercase">Net Profit</p>
                  </div>
                  <p className="text-3xl font-black text-green-400">₱{stats.totalProfit.toLocaleString()}</p>
                  <p className="text-[10px] font-bold text-slate-400 mt-2">Margin: {((stats.totalProfit / stats.total) * 100 || 0).toFixed(1)}%</p>
                </div>
             </div>

             <div className="grid grid-cols-1 lg:grid-cols-2 gap-8">
               <div className="bg-white p-10 rounded-[2.5rem] border border-slate-200 shadow-sm">
                  <div className="flex justify-between items-center mb-10">
                    <h3 className="text-xs font-black text-slate-400 uppercase tracking-widest">Profit by Game</h3>
                    <button onClick={generateAIInsights} disabled={aiLoading} className="text-red-700 text-xs font-black uppercase flex items-center gap-2 hover:opacity-70">
                      {aiLoading ? <Clock className="animate-spin w-3 h-3" /> : <Sparkles className="w-3 h-3" />} AI Analyze
                    </button>
                  </div>
                  <div className="space-y-6">
                    {categories.map(cat => {
                      const catOrders = orders.filter(o => o.category === cat.name);
                      const rev = catOrders.reduce((a, b) => a + Number(b.amount), 0);
                      const cost = catOrders.reduce((a, b) => a + Number(b.cost || 0), 0);
                      const profit = rev - cost;
                      const percentage = (profit / stats.totalProfit) * 100 || 0;
                      if (rev === 0) return null;
                      return (
                        <div key={cat.id} className="space-y-2">
                          <div className="flex justify-between items-end">
                            <p className="text-xs font-black text-slate-800 uppercase">{cat.name}</p>
                            <p className="text-[11px] font-black text-green-600">₱{profit.toLocaleString()}</p>
                          </div>
                          <div className="h-2 bg-slate-50 rounded-full overflow-hidden">
                            <div className="h-full bg-green-500" style={{ width: `${Math.max(0, percentage)}%` }}></div>
                          </div>
                        </div>
                      );
                    })}
                  </div>
               </div>

               <div className="bg-white p-10 rounded-[2.5rem] border border-slate-200 shadow-sm">
                  <h3 className="text-sm font-black text-slate-400 uppercase mb-8 tracking-widest">Recent Activity</h3>
                  <div className="space-y-3">
                    {orders.slice(0, 5).map(o => {
                       const profit = Number(o.amount) - Number(o.cost || 0);
                       return (
                         <div key={o.id} className="flex justify-between items-center p-4 bg-slate-50 rounded-2xl border border-slate-100">
                            <div>
                              <p className="text-[9px] font-black text-slate-400 uppercase">{o.date}</p>
                              <p className="font-bold text-slate-800 text-xs">{o.customerName}</p>
                            </div>
                            <div className={`text-right font-black text-sm ${profit >= 0 ? 'text-green-600' : 'text-red-600'}`}>
                              ₱{profit.toLocaleString()}
                            </div>
                         </div>
                       )
                    })}
                  </div>
               </div>
             </div>
          </div>
        ) : activeTab === 'categories' ? (
          <div className="grid grid-cols-1 md:grid-cols-2 gap-8">
             <div className="bg-white p-8 rounded-[2.5rem] border border-slate-200 h-fit shadow-sm">
              <h2 className="text-xl font-black mb-6 flex items-center gap-2"><LayoutGrid className="text-red-700" /> Games</h2>
              <form onSubmit={handleAddCategory} className="flex gap-3">
                <input type="text" placeholder="Add Game Title..." value={newCategory} onChange={e => setNewCategory(e.target.value)} className="flex-1 p-5 bg-slate-50 border border-slate-200 rounded-2xl font-bold outline-none" required />
                <button type="submit" className="bg-red-700 text-white p-5 rounded-2xl shadow-lg"><Plus /></button>
              </form>
            </div>
            <div className="grid grid-cols-1 sm:grid-cols-2 gap-4">
              {categories.map(c => (
                <div key={c.id} className="bg-white p-5 rounded-2xl border border-slate-200 flex justify-between items-center group shadow-sm">
                  <div className="flex items-center gap-3">
                    <Tag className="w-4 h-4 text-red-700" />
                    <span className="font-black text-slate-800 uppercase text-xs">{c.name}</span>
                  </div>
                  {c.id !== 'default' && <button onClick={() => deleteCategory(c.id)} className="text-slate-200 hover:text-red-500 transition"><Trash2 className="w-4 h-4" /></button>}
                </div>
              ))}
            </div>
          </div>
        ) : (
          /* Partners Tab */
          <div className="grid grid-cols-1 lg:grid-cols-3 gap-8">
            <div className="bg-white p-8 rounded-[2.5rem] border border-slate-200 h-fit shadow-sm">
              <h2 className="text-xl font-black mb-6 flex items-center gap-2"><UserPlus className="text-red-700" /> Partners</h2>
              <form onSubmit={handleAddReseller} className="space-y-4">
                <input type="text" placeholder="Partner Name" value={newReseller.name} onChange={e => setNewReseller({...newReseller, name: e.target.value})} className="w-full p-5 bg-slate-50 border border-slate-200 rounded-2xl font-bold outline-none" required />
                <input type="text" placeholder="Initials (e.g. TR)" maxLength="4" value={newReseller.initials} onChange={e => setNewReseller({...newReseller, initials: e.target.value.toUpperCase()})} className="w-full p-5 bg-slate-50 border border-slate-200 rounded-2xl font-bold outline-none uppercase" required />
                <button type="submit" className="w-full bg-red-700 text-white font-black py-5 rounded-2xl shadow-xl">Add Partner</button>
              </form>
            </div>
            <div className="lg:col-span-2 grid grid-cols-1 sm:grid-cols-2 gap-4">
              {resellers.map(r => (
                <div key={r.id} className="bg-white p-6 rounded-3xl border border-slate-200 flex justify-between items-center shadow-sm group">
                  <div className="flex items-center gap-4">
                    <div className="bg-slate-900 text-white w-14 h-14 rounded-2xl flex items-center justify-center font-black text-xl">{r.initials}</div>
                    <div>
                      <p className="font-black text-slate-900 text-lg">{r.name}</p>
                      <p className="text-[10px] font-bold text-slate-400 uppercase">Partner Code: {r.initials}</p>
                    </div>
                  </div>
                  <div className="text-right">
                    <p className="text-[10px] font-black text-slate-400 uppercase">Sales</p>
                    <p className="text-2xl font-black text-red-700">{orders.filter(o => o.resellerId === r.id).length}</p>
                  </div>
                </div>
              ))}
            </div>
          </div>
        )}
      </main>

      <footer className="fixed bottom-0 left-0 right-0 bg-white/80 backdrop-blur-xl border-t border-slate-200 p-6 z-30 shadow-2xl">
        <div className="max-w-7xl mx-auto flex justify-between items-center">
          <div className="flex items-center gap-6">
            <div className="text-left">
              <p className="text-[9px] font-black text-slate-400 uppercase leading-none mb-1">Portfolio Profit</p>
              <p className={`text-sm font-black ${stats.totalProfit >= 0 ? 'text-green-600' : 'text-red-600'}`}>
                ₱{stats.totalProfit.toLocaleString()}
              </p>
            </div>
            <div className="h-8 w-px bg-slate-200"></div>
            <div className="text-left">
              <p className="text-[9px] font-black text-slate-400 uppercase leading-none mb-1">Capital Used</p>
              <p className="text-sm font-black text-slate-500">₱{stats.totalCost.toLocaleString()}</p>
            </div>
          </div>
          <div className="bg-red-700 px-6 py-3 rounded-2xl text-white font-black text-[10px] uppercase tracking-widest flex items-center gap-2">
            <Shield className="w-3 h-3" /> Ledger Secure
          </div>
        </div>
      </footer>
    </div>
  );
};

export default App;
