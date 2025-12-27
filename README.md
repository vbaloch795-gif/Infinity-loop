# Infinity-loof intertainment for people's 
import React, { useState, useEffect, useRef, useCallback } from 'react';

const CHANNELS = ['memes', 'oddlysatisfying', 'natureisfuckinglit', 'aww', 'gaming', 'spaceporn', 'architecture'];

export default function App() {
  const [items, setItems] = useState([]);
  const [subreddit, setSubreddit] = useState('memes');
  const [lastId, setLastId] = useState(null);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState(null);
  const [searchQuery, setSearchQuery] = useState('');
  const [isMuted, setIsMuted] = useState(true);
  const [downloadingId, setDownloadingId] = useState(null);
  const [mousePos, setMousePos] = useState({ x: 0, y: 0 });

  const [dataSaver, setDataSaver] = useState(() => localStorage.getItem('data_saver') === 'true');
  const [liked, setLiked] = useState(() => {
    const saved = localStorage.getItem('loop_vault');
    return saved ? new Map(JSON.parse(saved)) : new Map();
  });

  useEffect(() => {
    localStorage.setItem('loop_vault', JSON.stringify(Array.from(liked.entries())));
    localStorage.setItem('data_saver', dataSaver);
  }, [liked, dataSaver]);

  const fetchContent = useCallback(async (isNewChannel = false) => {
    if (loading) return;
    setLoading(true);
    setError(null);
    try {
      const after = isNewChannel ? '' : (lastId ? `&after=${lastId}` : '');
      const res = await fetch(`https://www.reddit.com/r/${subreddit}/hot.json?limit=12${after}`);
      if (!res.ok) throw new Error("Subreddit not found");
      const json = await res.json();
      const newItems = json.data.children.map(c => {
        const d = c.data;
        let type = d.is_video ? 'video' : (d.url.includes('.gif') ? 'gif' : 'image');
        return { id: d.id, title: d.title, url: d.is_video ? d.media.reddit_video.fallback_url : d.url, author: d.author, type, permalink: `https://reddit.com${d.permalink}` };
      }).filter(i => i.url && !i.url.includes('gallery'));
      setItems(prev => isNewChannel ? newItems : [...prev, ...newItems]);
      setLastId(json.data.after);
    } catch (e) { setError(e.message); }
    setLoading(false);
  }, [subreddit, lastId, loading]);

  const handleDownload = async (item) => {
    setDownloadingId(item.id);
    try {
      const res = await fetch(item.url);
      const blob = await res.blob();
      const bUrl = window.URL.createObjectURL(blob);
      const link = document.createElement('a');
      link.href = bUrl;
      link.download = `${item.id}.mp4`;
      document.body.appendChild(link);
      link.click();
      link.remove();
    } catch (e) { window.open(item.url, '_blank'); }
    setDownloadingId(null);
  };

  useEffect(() => {
    fetchContent(true);
    let lastShake = 0;
    const handleMotion = (e) => {
      const { x, y, z } = e.accelerationIncludingGravity;
      const acc = Math.sqrt(x*x + y*y + z*z);
      if (acc > 25 && Date.now() - lastShake > 1500) {
        lastShake = Date.now();
        if (navigator.vibrate) navigator.vibrate(50);
        
        setSubreddit(CHANNELS[Math.floor(Math.random()*CHANNELS.length)]);
      }
    };
    window.addEventListener('devicemotion', handleMotion);
    return () => window.removeEventListener('devicemotion', handleMotion);
  }, [subreddit]);

  const observer = useRef();
  const lastElementRef = useCallback(node => {
    if (loading) return;
    if (observer.current) observer.current.disconnect();
    observer.current = new IntersectionObserver(entries => {
      if (entries[0].isIntersecting && lastId) fetchContent();
    });
    if (node) observer.current.observe(node);
  }, [loading, lastId, fetchContent]);

  return (
    <div onMouseMove={(e) => setMousePos({ x: e.clientX, y: e.clientY })} style={containerStyle}>
      <nav style={navStyle}>
        <h1 style={logoStyle}>∞ LOOP</h1>
        <div style={channelRow}>
          {CHANNELS.map(c => (
            <button key={c} onClick={() => setSubreddit(c)} style={{...btnStyle, color: subreddit === c ? '#00f2ff' : '#666'}}>
              {c.toUpperCase()}
            </button>
          ))}
          <button onClick={() => setDataSaver(!dataSaver)} style={iconBtn}>{dataSaver ? '📉' : '📊'}</button>
          <button onClick={() => setIsMuted(!isMuted)} style={iconBtn}>{isMuted ? '🔇' : '🔊'}</button>
        </div>
      </nav>
      <div style={gridStyle}>
        {items.map((item, idx) => (
          <div key={`${item.id}-${idx}`} ref={items.length === idx + 1 ? lastElementRef : null} style={cardStyle}>
            {item.type === 'video' && !dataSaver ? (
              <video src={item.url} muted={isMuted} autoPlay loop playsInline style={mediaStyle} />
            ) : ( <img src={item.url} style={mediaStyle} loading="lazy" alt="" /> )}
            <div style={overlayStyle}>
              <div style={{flex: 1}}>
                <p style={titleStyle}>{item.title}</p>
                <span style={authorStyle}>u/{item.author}</span>
              </div>
              <button onClick={() => handleDownload(item)} style={iconBtn}>
                {downloadingId === item.id ? '⏳' : '📥'}
              </button>
            </div>
          </div>
        ))}
      </div>
    </div>
  );
}

// Minimal Styles for the App
const containerStyle = { backgroundColor: '#000', color: '#fff', minHeight: '100dvh', fontFamily: 'sans-serif' };
const navStyle = { position: 'sticky', top: 0, zIndex: 100, background: 'rgba(0,0,0,0.9)', padding: '10px', borderBottom: '1px solid #222' };
const logoStyle = { textAlign: 'center', color: '#00f2ff', margin: '0 0 10px 0' };
const channelRow = { display: 'flex', justifyContent: 'center', gap: '10px' };
const gridStyle = { display: 'grid', gridTemplateColumns: 'repeat(auto-fill, minmax(300px, 1fr))', gap: '10px', padding: '10px' };
const cardStyle = { position: 'relative', height: '450px', borderRadius: '15px', overflow: 'hidden', background: '#111' };
const mediaStyle = { width: '100%', height: '100%', objectFit: 'cover' };
const overlayStyle = { position: 'absolute', bottom: 0, width: '100%', padding: '15px', background: 'linear-gradient(transparent, black)' };
const titleStyle = { fontSize: '0.8rem', margin: 0 };
const authorStyle = { fontSize: '0.7rem', color: '#00f2ff' };
const btnStyle = { background: 'none', border: 'none', cursor: 'pointer', fontSize: '0.7rem' };
const iconBtn = { background: 'none', border: 'none', cursor: 'pointer', fontSize: '1.1rem' };

