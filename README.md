# ```html
<!DOCTYPE html>
<html lang="en" class="dark">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>DevConnect - Peer Code Review Platform</title>
  <!-- Tailwind CSS via CDN -->
  <script src="https://cdn.tailwindcss.com"></script>
  <script>
    tailwind.config = {
      darkMode: 'class',
      theme: {
        extend: {
          colors: {
            devDark: '#0d1117',
            devCanvas: '#161b22',
            devBorder: '#30363d',
            devCard: '#21262d',
            devAccent: '#238636',
            devAccentHover: '#2ea043',
            devBlue: '#58a6ff',
            devPurple: '#bc8cff',
            devYellow: '#d29922',
            devRed: '#f85149'
          },
          fontFamily: {
            sans: ['-apple-system', 'BlinkMacSystemFont', '"Segoe UI"', 'Roboto', 'Helvetica', 'Arial', 'sans-serif'],
            mono: ['ui-monospace', 'SFMono-Regular', 'SF Mono', 'Menlo', 'Consolas', 'Liberation Mono', 'monospace']
          }
        }
      }
    }
  </script>
  <!-- FontAwesome Icons -->
  <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.4.0/css/all.min.css">
  <!-- React & ReactDOM CDN -->
  <script src="https://unpkg.com/react@18/umd/react.development.js" crossorigin></script>
  <script src="https://unpkg.com/react-dom@18/umd/react-dom.development.js" crossorigin></script>
  <!-- Babel Standalone for JSX compilation -->
  <script src="https://unpkg.com/@babel/standalone/babel.min.js"></script>

  <style>
    /* Custom scrollbars */
    ::-webkit-scrollbar {
      width: 8px;
      height: 8px;
    }
    ::-webkit-scrollbar-track {
      background: #0d1117;
    }
    ::-webkit-scrollbar-thumb {
      background: #30363d;
      border-radius: 4px;
    }
    ::-webkit-scrollbar-thumb:hover {
      background: #484f58;
    }
    
    .code-line:hover {
      background-color: rgba(56, 139, 253, 0.1);
    }
    .line-number {
      user-select: none;
    }
  </style>
</head>
<body class="bg-devDark text-gray-200 antialiased min-h-screen font-sans selection:bg-blue-600 selection:text-white">
  <div id="root"></div>

  <script type="text/babel">
    const { useState, useEffect, useMemo } = React;

    // --- MOCK INITIAL DATA ---
    const INITIAL_REVIEWS = [
      {
        id: "rev-1",
        title: "Fix memory leak in custom React useWebSocket hook",
        description: "Experiencing an unbounded event listener accumulation when the component re-renders rapidly during high socket traffic. Need help verifying cleanup sequence and dependency array logic.",
        author: {
          name: "alex_dev",
          avatar: "https://images.unsplash.com/photo-1534528741775-53994a69daeb?w=100&auto=format&fit=crop&q=80",
          reputation: 1420,
          badge: "Senior Contributor"
        },
        language: "javascript",
        tags: ["React", "Hooks", "WebSockets", "Performance"],
        status: "Open", // Open, Approved, Changes Requested
        prUrl: "https://github.com/org/repo/pull/42",
        createdAt: "2 hours ago",
        upvotes: 18,
        code: `import { useEffect, useRef, useState } from 'react';

export function useWebSocket(url, options = {}) {
  const [socket, setSocket] = useState(null);
  const [lastMessage, setLastMessage] = useState(null);
  const listenersRef = useRef([]);

  useEffect(() => {
    const ws = new WebSocket(url);
    setSocket(ws);

    ws.onmessage = (event) => {
      setLastMessage(event.data);
      // BUG: Appends listeners endlessly without deduplication
      listenersRef.current.forEach(cb => cb(event.data));
    };

    return () => {
      // Missing proper close or cleanup logic here!
      ws.close();
    };
  }, [url]);

  const addListener = (cb) => {
    listenersRef.current.push(cb);
  };

  return { socket, lastMessage, addListener };
}`,
        comments: [
          {
            id: "c1",
            author: {
              name: "sarah_code",
              avatar: "https://images.unsplash.com/photo-1494790108377-be9c29b29330?w=100&auto=format&fit=crop&q=80",
              reputation: 2890,
              badge: "Core Reviewer"
            },
            line: 14,
            text: "Your `listenersRef.current` keeps growing on every function register. You should return an un-subscribe function from `addListener` so components can detach listener on unmount.",
            suggestedCode: `  const addListener = (cb) => {
    listenersRef.current.push(cb);
    return () => {
      listenersRef.current = listenersRef.current.filter(fn => fn !== cb);
    };
  };`,
            createdAt: "1 hour ago",
            upvotes: 8
          },
          {
            id: "c2",
            author: {
              name: "tech_guru",
              avatar: "https://images.unsplash.com/photo-1507003211169-0a1dd7228f2d?w=100&auto=format&fit=crop&q=80",
              reputation: 3450,
              badge: "Maintainer"
            },
            line: null,
            text: "Overall clean approach. Consider wrapping callback execution inside `try/catch` to avoid one faulty listener crashing the websocket frame handler.",
            suggestedCode: null,
            createdAt: "30 mins ago",
            upvotes: 4
          }
        ]
      },
      {
        id: "rev-2",
        title: "FastAPI JWT Authentication Middleware optimization",
        description: "Refactored token decode logic to leverage async Redis cache for session validation. Please check for potential race conditions during token refresh.",
        author: {
          name: "marco_py",
          avatar: "https://images.unsplash.com/photo-1500648767791-00dcc994a43e?w=100&auto=format&fit=crop&q=80",
          reputation: 890,
          badge: "Contributor"
        },
        language: "python",
        tags: ["FastAPI", "Python", "JWT", "Security"],
        status: "Approved",
        prUrl: "https://github.com/fastapi-app/backend/pull/108",
        createdAt: "5 hours ago",
        upvotes: 24,
        code: `from fastapi import Request, HTTPException
from fastapi.security import HTTPBearer
import jwt
import aioredis

class JWTAuthMiddleware(HTTPBearer):
    def __init__(self, redis_url: str, secret_key: str):
        super().__init__()
        self.secret_key = secret_key
        self.redis_url = redis_url

    async def __call__(self, request: Request):
        credentials = await super().__call__(request)
        token = credentials.credentials
        
        try:
            payload = jwt.decode(token, self.secret_key, algorithms=["HS256"])
            user_id = payload.get("sub")
            
            # Check revocation list in Redis
            r = await aioredis.from_url(self.redis_url)
            is_blacklisted = await r.get(f"bl_{token}")
            if is_blacklisted:
                raise HTTPException(status_code=401, detail="Token revoked")
                
            return payload
        except jwt.PyJWTError:
            raise HTTPException(status_code=403, detail="Invalid token signature")`,
        comments: [
          {
            id: "c3",
            author: {
              name: "sec_watcher",
              avatar: "https://images.unsplash.com/photo-1522075469751-3a6694fb2f61?w=100&auto=format&fit=crop&q=80",
              reputation: 4120,
              badge: "Security Lead"
            },
            line: 23,
            text: "Creating a new Redis connection client per request (`aioredis.from_url`) inside the middleware payload function causes connection pool exhaustion! Use a dependency-injected connection pool singleton.",
            suggestedCode: `            # Inject pool instead of connecting on every HTTP request
            is_blacklisted = await request.app.state.redis.get(f"bl_{token}")`,
            createdAt: "3 hours ago",
            upvotes: 12
          }
        ]
      },
      {
        id: "rev-3",
        title: "Rust Lock-free Concurrent Task Worker Pool",
        description: "Implemented a work-stealing queue for multi-threaded background job distribution. Looking for advice on unsafe block guarantees.",
        author: {
          name: "ferris_rust",
          avatar: "https://images.unsplash.com/photo-1570295999919-56ceb5ecca61?w=100&auto=format&fit=crop&q=80",
          reputation: 3100,
          badge: "Rustacean"
        },
        language: "rust",
        tags: ["Rust", "Concurrency", "LockFree", "Performance"],
        status: "Changes Requested",
        prUrl: "",
        createdAt: "1 day ago",
        upvotes: 31,
        code: `use std::sync::atomic::{AtomicPtr, Ordering};
use std::ptr;

pub struct TaskNode<T> {
    pub value: Option<T>,
    pub next: AtomicPtr<TaskNode<T>>,
}

pub struct LockFreeQueue<T> {
    head: AtomicPtr<TaskNode<T>>,
    tail: AtomicPtr<TaskNode<T>>,
}

impl<T> LockFreeQueue<T> {
    pub fn new() -> Self {
        let dummy = Box::into_raw(Box::new(TaskNode {
            value: None,
            next: AtomicPtr::new(ptr::null_mut()),
        }));
        LockFreeQueue {
            head: AtomicPtr::new(dummy),
            tail: AtomicPtr::new(dummy),
        }
    }

    pub fn push(&self, value: T) {
        let new_node = Box::into_raw(Box::new(TaskNode {
            value: Some(value),
            next: AtomicPtr::new(ptr::null_mut()),
        }));

        loop {
            let tail = self.tail.load(Ordering::Acquire);
            let next = unsafe { (*tail).next.load(Ordering::Acquire) };

            if next.is_null() {
                if unsafe { (*tail).next.compare_exchange(
                    ptr::null_mut(),
                    new_node,
                    Ordering::Release,
                    Ordering::Relaxed
                ).is_ok() } {
                    let _ = self.tail.compare_exchange(
                        tail,
                        new_node,
                        Ordering::Release,
                        Ordering::Relaxed
                    );
                    break;
                }
            }
        }
    }
}`,
        comments: []
      }
    ];

    // Helper: Simple Syntax Highlighting Simulation
    const highlightSyntax = (code, language) => {
      if (!code) return '';
      // Escape HTML
      let escaped = code
        .replace(/&/g, '&amp;')
        .replace(/</g, '&lt;')
        .replace(/>/g, '&gt;');

      // Syntax regex patterns
      const keywords = /\b(const|let|var|function|return|export|import|from|async|await|if|else|try|catch|class|def|self|pub|struct|fn|use|impl|loop|let|mut|match|unsafe|new|super|type|select|from|where|insert|into|values)\b/g;
      const strings = /(["'])(?:(?=(\\?))\2.)*?\1|(`[\s\S]*?`)/g;
      const comments = /(\/\/[^\n]*|\/\*[\s\S]*?\*\/|#[^\n]*)/g;
      const numbers = /\b(\d+)\b/g;
      const functions = /\b([a-zA-Z_][a-zA-Z0-9_]*)(?=\()/g;

      return escaped
        .replace(comments, '<span class="text-gray-500 italic">$1</span>')
        .replace(strings, '<span class="text-emerald-400">$1</span>')
        .replace(keywords, '<span class="text-purple-400 font-semibold">$1</span>')
        .replace(functions, '<span class="text-blue-400">$1</span>')
        .replace(numbers, '<span class="text-amber-400">$1</span>');
    };

    // Main App Component
    function App() {
      // User State (Simulated Auth)
      const [currentUser, setCurrentUser] = useState({
        isLoggedIn: true,
        name: "octocat_pro",
        avatar: "https://images.unsplash.com/photo-1535713875002-d1d0cf377fde?w=100&auto=format&fit=crop&q=80",
        reputation: 2150,
        badge: "Lead Reviewer"
      });

      // Data States
      const [reviews, setReviews] = useState(INITIAL_REVIEWS);
      const [selectedReviewId, setSelectedReviewId] = useState("rev-1");
      const [isModalOpen, setIsModalOpen] = useState(false);

      // Filter & Search States
      const [searchQuery, setSearchQuery] = useState("");
      const [selectedLang, setSelectedLang] = useState("All");
      const [selectedStatus, setSelectedStatus] = useState("All");
      const [sortBy, setSortBy] = useState("latest");

      // Review Detail state
      const [activeTab, setActiveTab] = useState("code"); // "code" or "overview"
      const [targetLineComment, setTargetLineComment] = useState(null);
      const [commentText, setCommentText] = useState("");
      const [suggestionCode, setSuggestionCode] = useState("");
      const [showSuggestionInput, setShowSuggestionInput] = useState(false);

      const activeReview = useMemo(() => {
        return reviews.find(r => r.id === selectedReviewId) || reviews[0];
      }, [reviews, selectedReviewId]);

      // Filtered Reviews
      const filteredReviews = useMemo(() => {
        return reviews.filter(r => {
          const matchesSearch = r.title.toLowerCase().includes(searchQuery.toLowerCase()) ||
                                r.description.toLowerCase().includes(searchQuery.toLowerCase()) ||
                                r.tags.some(t => t.toLowerCase().includes(searchQuery.toLowerCase()));
          const matchesLang = selectedLang === "All" || r.language.toLowerCase() === selectedLang.toLowerCase();
          const matchesStatus = selectedStatus === "All" || r.status === selectedStatus;
          return matchesSearch && matchesLang && matchesStatus;
        }).sort((a, b) => {
          if (sortBy === "latest") return b.id.localeCompare(a.id);
          if (sortBy === "upvotes") return b.upvotes - a.upvotes;
          if (sortBy === "comments") return b.comments.length - a.comments.length;
          return 0;
        });
      }, [reviews, searchQuery, selectedLang, selectedStatus, sortBy]);

      // Handlers
      const handleLoginToggle = () => {
        if (currentUser.isLoggedIn) {
          setCurrentUser(prev => ({ ...prev, isLoggedIn: false }));
        } else {
          setCurrentUser({
            isLoggedIn: true,
            name: "octocat_pro",
            avatar: "https://images.unsplash.com/photo-1535713875002-d1d0cf377fde?w=100&auto=format&fit=crop&q=80",
            reputation: 2150,
            badge: "Lead Reviewer"
          });
        }
      };

      const handleAddReview = (newRev) => {
        const reviewObject = {
          ...newRev,
          id: `rev-${Date.now()}`,
          author: {
            name: currentUser.name,
            avatar: currentUser.avatar,
            reputation: currentUser.reputation,
            badge: currentUser.badge
          },
          status: "Open",
          createdAt: "Just now",
          upvotes: 1,
          comments: []
        };
        setReviews([reviewObject, ...reviews]);
        setSelectedReviewId(reviewObject.id);
        setIsModalOpen(false);
      };

      const handleUpvoteReview = (id, e) => {
        e.stopPropagation();
        setReviews(reviews.map(r => r.id === id ? { ...r, upvotes: r.upvotes + 1 } : r));
      };

      const handleStatusChange = (newStatus) => {
        setReviews(reviews.map(r => r.id === activeReview.id ? { ...r, status: newStatus } : r));
      };

      const handleAddComment = (e) => {
        e.preventDefault();
        if (!commentText.trim()) return;

        const newComment = {
          id: `c-${Date.now()}`,
          author: {
            name: currentUser.name,
            avatar: currentUser.avatar,
            reputation: currentUser.reputation,
            badge: currentUser.badge
          },
          line: targetLineComment,
          text: commentText,
          suggestedCode: showSuggestionInput && suggestionCode.trim() ? suggestionCode : null,
          createdAt: "Just now",
          upvotes: 0
        };

        setReviews(reviews.map(r => {
          if (r.id === activeReview.id) {
            return {
              ...r,
              comments: [...r.comments, newComment]
            };
          }
          return r;
        }));

        setCommentText("");
        setSuggestionCode("");
        setShowSuggestionInput(false);
        setTargetLineComment(null);
      };

      const handleUpvoteComment = (commentId) => {
        setReviews(reviews.map(r => {
          if (r.id === activeReview.id) {
            return {
              ...r,
              comments: r.comments.map(c => c.id === commentId ? { ...c, upvotes: c.upvotes + 1 } : c)
            };
          }
          return r;
        }));
      };

      return (
        <div class="flex flex-col h-screen overflow-hidden bg-devDark text-gray-100">
          
          {/* HEADER BAR */}
          <header class="h-16 border-b border-devBorder bg-devCanvas px-4 md:px-6 flex items-center justify-between flex-shrink-0 z-20">
            <div class="flex items-center space-x-3">
              <div class="bg-gradient-to-tr from-green-500 to-blue-600 p-2 rounded-lg text-white font-bold text-xl flex items-center justify-center w-10 h-10 shadow-lg shadow-green-900/20">
                <i class="fa-solid fa-code-pull-request"></i>
              </div>
              <div>
                <span class="text-xl font-bold tracking-tight bg-clip-text text-transparent bg-gradient-to-r from-white via-gray-200 to-gray-400">
                  DevConnect
                </span>
                <span class="ml-2 text-xs px-2 py-0.5 rounded bg-devBorder text-devBlue font-mono">
                  v1.0-beta
                </span>
              </div>
            </div>

            {/* Header Right Action & User */}
            <div class="flex items-center space-x-4">
              <button 
                onClick={() => setIsModalOpen(true)}
                class="bg-devAccent hover:bg-devAccentHover text-white px-3.5 py-1.5 rounded-md text-sm font-medium flex items-center space-x-2 transition shadow-sm hover:shadow-green-950"
              >
                <i class="fa-solid fa-plus text-xs"></i>
                <span class="hidden sm:inline">Submit Request</span>
              </button>

              <div class="h-6 w-px bg-devBorder hidden sm:block"></div>

              {currentUser.isLoggedIn ? (
                <div class="flex items-center space-x-3">
                  <div class="flex items-center space-x-2 bg-devCard border border-devBorder px-3 py-1 rounded-full">
                    <img src={currentUser.avatar} alt="User" class="w-6 h-6 rounded-full ring-1 ring-devBlue" />
                    <span class="text-xs font-mono font-semibold text-gray-200">{currentUser.name}</span>
                    <span class="text-xs bg-amber-500/20 text-amber-300 px-1.5 py-0.5 rounded font-mono font-bold flex items-center space-x-1">
                      <i class="fa-solid fa-star text-[10px]"></i>
                      <span>{currentUser.reputation}</span>
                    </span>
                  </div>
                  <button 
                    onClick={handleLoginToggle} 
                    title="Logout"
                    class="text-gray-400 hover:text-devRed text-sm p-1 transition"
                  >
                    <i class="fa-solid fa-right-from-bracket"></i>
                  </button>
                </div>
              ) : (
                <button 
                  onClick={handleLoginToggle}
                  class="bg-devCard border border-devBorder hover:border-devBlue text-white px-3.5 py-1.5 rounded-md text-sm font-medium flex items-center space-x-2 transition"
                >
                  <i class="fa-brands fa-github text-base"></i>
                  <span>Sign in with GitHub</span>
                </button>
              )}
            </div>
          </header>

          {/* MAIN WORKSPACE BODY */}
          <div class="flex-1 flex overflow-hidden">

            {/* LEFT SIDEBAR: REQUEST FEED & FILTERS */}
            <div class="w-full md:w-96 border-r border-devBorder bg-devDark flex flex-col h-full flex-shrink-0">
              
              {/* Filter Controls */}
              <div class="p-4 border-b border-devBorder bg-devCanvas/50 space-y-3">
                {/* Search */}
                <div class="relative">
                  <i class="fa-solid fa-magnifying-glass absolute left-3 top-2.5 text-xs text-gray-400"></i>
                  <input
                    type="text"
                    placeholder="Search snippet title, tag..."
                    value={searchQuery}
                    onChange={(e) => setSearchQuery(e.target.value)}
                    class="w-full bg-devDark border border-devBorder focus:border-devBlue focus:outline-none rounded-md pl-8 pr-3 py-1.5 text-xs text-gray-200 placeholder-gray-500"
                  />
                </div>

                {/* Filter selects */}
                <div class="grid grid-cols-2 gap-2 text-xs">
                  <div>
                    <label class="block text-[10px] uppercase text-gray-400 font-mono mb-1">Language</label>
                    <select
                      value={selectedLang}
                      onChange={(e) => setSelectedLang(e.target.value)}
                      class="w-full bg-devCard border border-devBorder rounded py-1 px-2 text-gray-300 focus:outline-none"
                    >
                      <option value="All">All Languages</option>
                      <option value="javascript">JavaScript</option>
                      <option value="python">Python</option>
                      <option value="rust">Rust</option>
                      <option value="go">Go</option>
                      <option value="typescript">TypeScript</option>
                    </select>
                  </div>

                  <div>
                    <label class="block text-[10px] uppercase text-gray-400 font-mono mb-1">Status</label>
                    <select
                      value={selectedStatus}
                      onChange={(e) => setSelectedStatus(e.target.value)}
                      class="w-full bg-devCard border border-devBorder rounded py-1 px-2 text-gray-300 focus:outline-none"
                    >
                      <option value="All">All Statuses</option>
                      <option value="Open">Open</option>
                      <option value="Approved">Approved</option>
                      <option value="Changes Requested">Changes Requested</option>
                    </select>
                  </div>
                </div>

                {/* Sort selector */}
                <div class="flex items-center justify-between text-xs pt-1">
                  <span class="text-gray-400 text-[11px] font-mono">{filteredReviews.length} Reviews found</span>
                  <div class="flex items-center space-x-1">
                    <span class="text-gray-500 text-[11px]">Sort:</span>
                    <button 
                      onClick={() => setSortBy(sortBy === 'latest' ? 'upvotes' : 'latest')}
                      class="text-devBlue hover:underline text-[11px] font-mono uppercase"
                    >
                      {sortBy}
                    </button>
                  </div>
                </div>
              </div>

              {/* Review Feed Cards */}
              <div class="flex-1 overflow-y-auto divide-y divide-devBorder">
                {filteredReviews.length === 0 ? (
                  <div class="p-8 text-center text-gray-500 text-xs">
                    <i class="fa-solid fa-code-commit text-2xl mb-2 text-gray-600 block"></i>
                    No code review requests match your criteria.
                  </div>
                ) : (
                  filteredReviews.map(item => {
                    const isSelected = item.id === activeReview?.id;
                    return (
                      <div
                        key={item.id}
                        onClick={() => setSelectedReviewId(item.id)}
                        class={`p-4 cursor-pointer transition border-l-2 ${
                          isSelected 
                            ? 'bg-devCard border-devBlue' 
                            : 'hover:bg-devCanvas/60 border-transparent'
                        }`}
                      >
                        <div class="flex items-start justify-between mb-1.5">
                          <span class={`text-[10px] font-mono px-2 py-0.5 rounded-full font-semibold ${
                            item.status === 'Open' ? 'bg-emerald-500/10 text-emerald-400 border border-emerald-500/30' :
                            item.status === 'Approved' ? 'bg-blue-500/10 text-blue-400 border border-blue-500/30' :
                            'bg-amber-500/10 text-amber-400 border border-amber-500/30'
                          }`}>
                            {item.status}
                          </span>
                          <span class="text-[11px] text-gray-500 font-mono">{item.createdAt}</span>
                        </div>

                        <h3 class="text-sm font-semibold text-gray-100 line-clamp-2 mb-1.5 hover:text-devBlue">
                          {item.title}
                        </h3>

                        <p class="text-xs text-gray-400 line-clamp-2 mb-3">
                          {item.description}
                        </p>

                        {/* Tags */}
                        <div class="flex flex-wrap gap-1 mb-3">
                          {item.tags.map((t, idx) => (
                            <span key={idx} class="text-[10px] bg-devDark border border-devBorder text-gray-400 px-1.5 py-0.5 rounded">
                              #{t}
                            </span>
                          ))}
                        </div>

                        {/* Footer Card Info */}
                        <div class="flex items-center justify-between text-xs text-gray-400 pt-2 border-t border-devBorder/40">
                          <div class="flex items-center space-x-2">
                            <img src={item.author.avatar} class="w-4 h-4 rounded-full" />
                            <span class="text-[11px] text-gray-300 font-mono truncate max-w-[100px]">{item.author.name}</span>
                          </div>

                          <div class="flex items-center space-x-3 text-[11px]">
                            <button 
                              onClick={(e) => handleUpvoteReview(item.id, e)}
                              class="hover:text-amber-400 flex items-center space-x-1"
                            >
                              <i class="fa-regular fa-thumbs-up"></i>
                              <span>{item.upvotes}</span>
                            </button>

                            <span class="flex items-center space-x-1">
                              <i class="fa-regular fa-comment"></i>
                              <span>{item.comments.length}</span>
                            </span>
                          </div>
                        </div>
                      </div>
                    );
                  })
                )}
              </div>
            </div>

            {/* RIGHT WORKSPACE VIEW (Active Code Review) */}
            {activeReview ? (
              <div class="flex-1 flex flex-col h-full bg-devDark overflow-hidden">
                
                {/* Active Review Title Bar */}
                <div class="p-4 md:p-6 border-b border-devBorder bg-devCanvas flex-shrink-0">
                  <div class="flex flex-col md:flex-row md:items-center justify-between gap-4">
                    <div>
                      <div class="flex items-center space-x-3 mb-2">
                        <span class="text-xs px-2.5 py-1 rounded-md font-mono bg-devDark border border-devBorder text-devPurple font-bold">
                          {activeReview.language.toUpperCase()}
                        </span>
                        <h1 class="text-lg md:text-xl font-bold text-gray-100 tracking-tight">
                          {activeReview.title}
                        </h1>
                      </div>

                      <div class="flex items-center space-x-3 text-xs text-gray-400 font-mono">
                        <div class="flex items-center space-x-1.5">
                          <img src={activeReview.author.avatar} class="w-4 h-4 rounded-full" />
                          <span class="text-gray-200 font-semibold">{activeReview.author.name}</span>
                          <span class="text-[10px] bg-devBorder px-1.5 rounded text-devBlue">{activeReview.author.badge}</span>
                        </div>
                        <span>•</span>
                        <span>{activeReview.createdAt}</span>
                        {activeReview.prUrl && (
                          <>
                            <span>•</span>
                            <a href={activeReview.prUrl} target="_blank" rel="noreferrer" class="text-devBlue hover:underline flex items-center space-x-1">
                              <i class="fa-brands fa-github text-xs"></i>
                              <span>PR Link</span>
                            </a>
                          </>
                        )}
                      </div>
                    </div>

                    {/* Status Toggle buttons */}
                    <div class="flex items-center space-x-2">
                      <span class="text-xs text-gray-400 mr-1 hidden lg:inline">Review Verdict:</span>
                      <button
                        onClick={() => handleStatusChange("Approved")}
                        class={`px-3 py-1.5 rounded text-xs font-semibold flex items-center space-x-1 transition ${
                          activeReview.status === 'Approved' 
                            ? 'bg-emerald-600 text-white' 
                            : 'bg-devCard border border-devBorder hover:border-emerald-500 text-gray-300'
                        }`}
                      >
                        <i class="fa-solid fa-check text-[10px]"></i>
                        <span>Approve</span>
                      </button>

                      <button
                        onClick={() => handleStatusChange("Changes Requested")}
                        class={`px-3 py-1.5 rounded text-xs font-semibold flex items-center space-x-1 transition ${
                          activeReview.status === 'Changes Requested' 
                            ? 'bg-amber-600 text-white' 
                            : 'bg-devCard border border-devBorder hover:border-amber-500 text-gray-300'
                        }`}
                      >
                        <i class="fa-solid fa-triangle-exclamation text-[10px]"></i>
                        <span>Request Changes</span>
                      </button>
                    </div>
                  </div>

                  {/* Tabs */}
                  <div class="flex items-center space-x-6 mt-6 border-b border-devBorder/60 text-xs font-medium">
                    <button
                      onClick={() => setActiveTab("code")}
                      class={`pb-2 flex items-center space-x-2 border-b-2 transition ${
                        activeTab === 'code' ? 'border-devBlue text-devBlue font-semibold' : 'border-transparent text-gray-400 hover:text-gray-200'
                      }`}
                    >
                      <i class="fa-solid fa-code"></i>
                      <span>Code File ({activeReview.code.split('\n').length} lines)</span>
                    </button>

                    <button
                      onClick={() => setActiveTab("overview")}
                      class={`pb-2 flex items-center space-x-2 border-b-2 transition ${
                        activeTab === 'overview' ? 'border-devBlue text-devBlue font-semibold' : 'border-transparent text-gray-400 hover:text-gray-200'
                      }`}
                    >
                      <i class="fa-solid fa-comments"></i>
                      <span>Discussion ({activeReview.comments.length})</span>
                    </button>
                  </div>
                </div>

                {/* Tab 1: CODE EDITOR & INLINE COMMENTS */}
                {activeTab === 'code' && (
                  <div class="flex-1 flex flex-col lg:flex-row overflow-hidden">
                    
                    {/* Code Display Area */}
                    <div class="flex-1 overflow-auto bg-devDark font-mono text-xs p-4 border-r border-devBorder">
                      <div class="mb-2 text-gray-500 text-[11px] flex justify-between items-center">
                        <span>Click line number to attach code comment</span>
                        <span>Syntax: {activeReview.language}</span>
                      </div>

                      <div class="border border-devBorder rounded-lg overflow-hidden bg-devCanvas">
                        <table class="w-full border-collapse">
                          <tbody>
                            {activeReview.code.split('\n').map((line, idx) => {
                              const lineNum = idx + 1;
                              const hasComments = activeReview.comments.some(c => c.line === lineNum);

                              return (
                                <tr key={idx} class="code-line group transition">
                                  {/* Line Number Button */}
                                  <td 
                                    onClick={() => setTargetLineComment(targetLineComment === lineNum ? null : lineNum)}
                                    class={`w-12 text-right pr-3 py-0.5 text-gray-600 font-mono text-[11px] line-number cursor-pointer border-r border-devBorder select-none group-hover:text-devBlue group-hover:bg-devCard ${
                                      targetLineComment === lineNum ? 'bg-devBlue/20 text-devBlue font-bold' : ''
                                    }`}
                                  >
                                    {hasComments && <i class="fa-solid fa-comment text-[9px] text-devBlue mr-1"></i>}
                                    {lineNum}
                                  </td>
                                  
                                  {/* Code Text */}
                                  <td class="pl-4 py-0.5 whitespace-pre font-mono text-gray-200">
                                    <span 
                                      dangerouslySetInnerHTML={{
                                        __html: highlightSyntax(line, activeReview.language)
                                      }}
                                    />
                                  </td>
                                </tr>
                              );
                            })}
                          </tbody>
                        </table>
                      </div>
                    </div>

                    {/* Inline Comment Inspector Panel */}
                    <div class="w-full lg:w-96 border-t lg:border-t-0 border-devBorder bg-devCanvas flex flex-col h-full flex-shrink-0">
                      
                      <div class="p-3 border-b border-devBorder bg-devDark/50 flex items-center justify-between">
                        <span class="text-xs font-semibold text-gray-300 font-mono flex items-center space-x-2">
                          <i class="fa-solid fa-layer-group text-devBlue"></i>
                          <span>{targetLineComment ? `Line ${targetLineComment} Comments` : 'Code Reviews & Feedback'}</span>
                        </span>
                        {targetLineComment && (
                          <button 
                            onClick={() => setTargetLineComment(null)}
                            class="text-[11px] text-devBlue hover:underline"
                          >
                            View All
                          </button>
                        )}
                      </div>

                      {/* Comment Thread List */}
                      <div class="flex-1 overflow-y-auto p-4 space-y-4">
                        {activeReview.comments
                          .filter(c => targetLineComment === null || c.line === targetLineComment)
                          .length === 0 ? (
                            <div class="text-center py-8 text-gray-500 text-xs">
                              <i class="fa-regular fa-comment-dots text-2xl mb-2 block text-gray-600"></i>
                              {targetLineComment 
                                ? `No inline comments on line ${targetLineComment} yet.` 
                                : 'No code comments attached yet. Submit one below!'}
                            </div>
                          ) : (
                            activeReview.comments
                              .filter(c => targetLineComment === null || c.line === targetLineComment)
                              .map(comment => (
                                <div key={comment.id} class="bg-devDark border border-devBorder rounded-lg p-3 space-y-2 text-xs">
                                  <div class="flex items-center justify-between">
                                    <div class="flex items-center space-x-2">
                                      <img src={comment.author.avatar} class="w-4 h-4 rounded-full" />
                                      <span class="font-semibold text-gray-200">{comment.author.name}</span>
                                      {comment.line && (
                                        <span class="text-[10px] bg-devBlue/10 text-devBlue border border-devBlue/20 px-1.5 py-0.5 rounded font-mono">
                                          L{comment.line}
                                        </span>
                                      )}
                                    </div>
                                    <span class="text-[10px] text-gray-500">{comment.createdAt}</span>
                                  </div>

                                  <p class="text-gray-300 leading-relaxed">{comment.text}</p>

                                  {/* Code Suggestion Diff Box */}
                                  {comment.suggestedCode && (
                                    <div class="mt-2 border border-devBorder rounded overflow-hidden">
                                      <div class="bg-devCard px-2 py-1 text-[10px] text-gray-400 font-mono border-b border-devBorder flex items-center space-x-1">
                                        <i class="fa-solid fa-wand-magic-sparkles text-amber-400"></i>
                                        <span>Suggested Change</span>
                                      </div>
                                      <pre class="bg-emerald-950/20 p-2 text-[11px] font-mono text-emerald-300 overflow-x-auto">
                                        <code>{comment.suggestedCode}</code>
                                      </pre>
                                    </div>
                                  )}

                                  <div class="flex items-center justify-end pt-1">
                                    <button 
                                      onClick={() => handleUpvoteComment(comment.id)}
                                      class="text-[11px] text-gray-400 hover:text-amber-400 flex items-center space-x-1"
                                    >
                                      <i class="fa-regular fa-thumbs-up"></i>
                                      <span>{comment.upvotes}</span>
                                    </button>
                                  </div>
                                </div>
                              ))
                          )
                        }
                      </div>

                      {/* Add Comment Form */}
                      <form onSubmit={handleAddComment} class="p-3 border-t border-devBorder bg-devDark">
                        <div class="mb-2 flex items-center justify-between">
                          <span class="text-[11px] font-mono text-gray-400">
                            {targetLineComment ? `Attach to Line ${targetLineComment}` : 'General Code Feedback'}
                          </span>
                          <button
                            type="button"
                            onClick={() => setShowSuggestionInput(!showSuggestionInput)}
                            class={`text-[11px] font-mono flex items-center space-x-1 ${
                              showSuggestionInput ? 'text-amber-400' : 'text-gray-400 hover:text-gray-200'
                            }`}
                          >
                            <i class="fa-solid fa-code-compare"></i>
                            <span>{showSuggestionInput ? 'Remove Diff' : '+ Add Code Diff'}</span>
                          </button>
                        </div>

                        <textarea
                          rows="2"
                          placeholder="Write feedback..."
                          value={commentText}
                          onChange={(e) => setCommentText(e.target.value)}
                          class="w-full bg-devCard border border-devBorder focus:border-devBlue focus:outline-none rounded p-2 text-xs text-gray-200 placeholder-gray-500 mb-2"
                        ></textarea>

                        {showSuggestionInput && (
                          <div class="mb-2">
                            <label class="block text-[10px] font-mono text-amber-400 mb-1">Suggested Replacement Code:</label>
                            <textarea
                              rows="3"
                              placeholder="// Enter suggested code diff..."
                              value={suggestionCode}
                              onChange={(e) => setSuggestionCode(e.target.value)}
                              class="w-full bg-devDark font-mono border border-devBorder focus:border-amber-500 focus:outline-none rounded p-2 text-[11px] text-emerald-400 placeholder-gray-600"
                            ></textarea>
                          </div>
                        )}

                        <div class="flex justify-end">
                          <button
                            type="submit"
                            class="bg-devBlue hover:bg-blue-600 text-white px-3 py-1 rounded text-xs font-semibold transition"
                          >
                            Submit Comment
                          </button>
                        </div>
                      </form>
                    </div>
                  </div>
                )}

                {/* Tab 2: OVERVIEW & DESCRIPTION */}
                {activeTab === 'overview' && (
                  <div class="flex-1 overflow-y-auto p-6 space-y-6">
                    <div class="bg-devCanvas border border-devBorder rounded-xl p-6">
                      <h3 class="text-sm font-semibold text-gray-300 font-mono mb-2 uppercase tracking-wider">
                        Request Overview & Context
                      </h3>
                      <p class="text-sm text-gray-200 leading-relaxed whitespace-pre-line">
                        {activeReview.description}
                      </p>
                    </div>

                    <div class="bg-devCanvas border border-devBorder rounded-xl p-6">
                      <h3 class="text-sm font-semibold text-gray-300 font-mono mb-4 uppercase tracking-wider">
                        Review Activity Timeline
                      </h3>
                      <div class="space-y-4">
                        <div class="flex items-start space-x-3 text-xs">
                          <div class="w-7 h-7 rounded-full bg-devBlue/20 text-devBlue flex items-center justify-center flex-shrink-0 mt-0.5">
                            <i class="fa-solid fa-circle-info"></i>
                          </div>
                          <div>
                            <p class="text-gray-200">
                              <span class="font-semibold">{activeReview.author.name}</span> opened this review request.
                            </p>
                            <span class="text-[10px] text-gray-500 font-mono">{activeReview.createdAt}</span>
                          </div>
                        </div>

                        {activeReview.comments.map(c => (
                          <div key={c.id} class="flex items-start space-x-3 text-xs">
                            <img src={c.author.avatar} class="w-7 h-7 rounded-full flex-shrink-0 mt-0.5" />
                            <div>
                              <p class="text-gray-200">
                                <span class="font-semibold">{c.author.name}</span> commented {c.line ? `on line ${c.line}` : ''}
                              </p>
                              <p class="text-gray-400 mt-0.5">{c.text}</p>
                              <span class="text-[10px] text-gray-500 font-mono">{c.createdAt}</span>
                            </div>
                          </div>
                        ))}
                      </div>
                    </div>
                  </div>
                )}

              </div>
            ) : (
              <div class="flex-1 flex items-center justify-center text-gray-500 text-sm">
                Select a code review request from the feed to get started.
              </div>
            )}

          </div>

          {/* CREATE CODE REVIEW MODAL */}
          {isModalOpen && (
            <div class="fixed inset-0 z-50 bg-black/70 backdrop-blur-sm flex items-center justify-center p-4">
              <div class="bg-devCanvas border border-devBorder rounded-xl w-full max-w-2xl overflow-hidden shadow-2xl animate-fade-in">
                
                {/* Modal Header */}
                <div class="p-4 border-b border-devBorder flex items-center justify-between bg-devDark">
                  <div class="flex items-center space-x-2">
                    <i class="fa-solid fa-code-pull-request text-devBlue"></i>
                    <h2 class="text-sm font-bold text-gray-100 font-mono">Submit New Code Review Request</h2>
                  </div>
                  <button 
                    onClick={() => setIsModalOpen(false)}
                    class="text-gray-400 hover:text-white"
                  >
                    <i class="fa-solid fa-xmark"></i>
                  </button>
                </div>

                {/* Modal Form */}
                <form 
                  onSubmit={(e) => {
                    e.preventDefault();
                    const formData = new FormData(e.target);
                    handleAddReview({
                      title: formData.get('title'),
                      description: formData.get('description'),
                      language: formData.get('language'),
                      tags: formData.get('tags').split(',').map(t => t.trim()).filter(Boolean),
                      prUrl: formData.get('prUrl'),
                      code: formData.get('code')
                    });
                  }} 
                  class="p-6 space-y-4 max-h-[80vh] overflow-y-auto"
                >
                  <div>
                    <label class="block text-xs font-mono text-gray-300 mb-1">Title *</label>
                    <input
                      required
                      name="title"
                      type="text"
                      placeholder="e.g. Optimize SQL query indexing for user timeline"
                      class="w-full bg-devDark border border-devBorder focus:border-devBlue focus:outline-none rounded p-2 text-xs text-gray-200"
                    />
                  </div>

                  <div class="grid grid-cols-1 md:grid-cols-2 gap-4">
                    <div>
                      <label class="block text-xs font-mono text-gray-300 mb-1">Language *</label>
                      <select
                        name="language"
                        class="w-full bg-devDark border border-devBorder rounded p-2 text-xs text-gray-200 focus:outline-none"
                      >
                        <option value="javascript">JavaScript</option>
                        <option value="typescript">TypeScript</option>
                        <option value="python">Python</option>
                        <option value="rust">Rust</option>
                        <option value="go">Go</option>
                        <option value="sql">SQL</option>
                      </select>
                    </div>

                    <div>
                      <label class="block text-xs font-mono text-gray-300 mb-1">GitHub PR Link (Optional)</label>
                      <input
                        name="prUrl"
                        type="url"
                        placeholder="https://github.com/user/repo/pull/1"
                        class="w-full bg-devDark border border-devBorder focus:border-devBlue focus:outline-none rounded p-2 text-xs text-gray-200"
                      />
                    </div>
                  </div>

                  <div>
                    <label class="block text-xs font-mono text-gray-300 mb-1">Tags (Comma separated)</label>
                    <input
                      name="tags"
                      type="text"
                      placeholder="React, Performance, Database, Security"
                      class="w-full bg-devDark border border-devBorder focus:border-devBlue focus:outline-none rounded p-2 text-xs text-gray-200"
                    />
                  </div>

                  <div>
                    <label class="block text-xs font-mono text-gray-300 mb-1">Description / Problem Statement *</label>
                    <textarea
                      required
                      name="description"
                      rows="3"
                      placeholder="Describe what you need peer reviewers to focus on..."
                      class="w-full bg-devDark border border-devBorder focus:border-devBlue focus:outline-none rounded p-2 text-xs text-gray-200"
                    ></textarea>
                  </div>

                  <div>
                    <label class="block text-xs font-mono text-gray-300 mb-1">Code Snippet *</label>
                    <textarea
                      required
                      name="code"
                      rows="8"
                      placeholder="// Paste code snippet here..."
                      class="w-full bg-devDark font-mono border border-devBorder focus:border-devBlue focus:outline-none rounded p-2 text-xs text-gray-200"
                    ></textarea>
                  </div>

                  {/* Submit buttons */}
                  <div class="flex justify-end space-x-3 pt-3 border-t border-devBorder">
                    <button
                      type="button"
                      onClick={() => setIsModalOpen(false)}
                      class="px-4 py-1.5 rounded text-xs font-medium border border-devBorder text-gray-300 hover:bg-devCard"
                    >
                      Cancel
                    </button>
                    <button
                      type="submit"
                      class="px-4 py-1.5 rounded text-xs font-semibold bg-devAccent hover:bg-devAccentHover text-white shadow-md"
                    >
                      Publish Code Review Request
                    </button>
                  </div>
                </form>

              </div>
            </div>
          )}

        </div>
      );
    }

    // Render App
    const root = ReactDOM.createRoot(document.getElementById('root'));
    root.render(<App />);
  </script>
</body>
</html>
```
