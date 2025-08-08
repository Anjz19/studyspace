import React, { useState, useEffect } from 'react';
import { initializeApp } from 'firebase/app';
import { getAuth, signInWithCustomToken, signInAnonymously, onAuthStateChanged } from 'firebase/auth';
import { getFirestore, collection, addDoc, onSnapshot, query, orderBy, serverTimestamp } from 'firebase/firestore';

// Safely get Firebase config from the environment
const getFirebaseConfig = () => {
    try {
        if (typeof __firebase_config !== 'undefined') {
            return JSON.parse(__firebase_config);
        }
    } catch (e) {
        console.error("Failed to parse __firebase_config:", e);
    }
    return null;
};

// Safely get app ID from the environment
const getAppId = () => typeof __app_id !== 'undefined' ? __app_id : 'default-app-id';
const appId = getAppId();

const App = () => {
    const [lessons, setLessons] = useState([]);
    const [messages, setMessages] = useState([]);
    const [lessonTitle, setLessonTitle] = useState('');
    const [lessonContent, setLessonContent] = useState('');
    const [chatMessage, setChatMessage] = useState('');
    const [user, setUser] = useState(null);
    const [isAuthReady, setIsAuthReady] = useState(false);
    const [loading, setLoading] = useState(true);

    // useEffect for Firebase Initialization and Authentication
    useEffect(() => {
        const firebaseConfig = getFirebaseConfig();
        if (!firebaseConfig) {
            console.error("Firebase config is not available. Cannot initialize Firebase.");
            return;
        }

        const app = initializeApp(firebaseConfig);
        const auth = getAuth(app);
        
        // This listener ensures we're authenticated before trying to fetch data
        const unsubscribeAuth = onAuthStateChanged(auth, async (currentUser) => {
            if (currentUser) {
                setUser(currentUser);
            } else {
                try {
                    const token = typeof __initial_auth_token !== 'undefined' ? __initial_auth_token : null;
                    if (token) {
                        await signInWithCustomToken(auth, token);
                    } else {
                        await signInAnonymously(auth);
                    }
                } catch (error) {
                    console.error("Authentication error:", error);
                }
            }
            setIsAuthReady(true);
        });

        return () => unsubscribeAuth();
    }, []);

    // useEffect for Firestore Data Listeners (Lessons and Chat)
    useEffect(() => {
        if (!isAuthReady) {
            return;
        }

        const firebaseConfig = getFirebaseConfig();
        const app = initializeApp(firebaseConfig);
        const db = getFirestore(app);

        // Define collection paths for public data
        const lessonsCollectionPath = `artifacts/${appId}/public/data/lessons`;
        const chatCollectionPath = `artifacts/${appId}/public/data/chat`;
        
        // Setup listener for lessons
        const lessonsQuery = query(collection(db, lessonsCollectionPath));
        const unsubscribeLessons = onSnapshot(lessonsQuery, (querySnapshot) => {
            const lessonsData = [];
            querySnapshot.forEach((doc) => {
                lessonsData.push({ id: doc.id, ...doc.data() });
            });
            // Sort lessons by creation time to show the newest first
            lessonsData.sort((a, b) => (b.createdAt || 0) - (a.createdAt || 0));
            setLessons(lessonsData);
            setLoading(false);
        }, (error) => {
            console.error("Error fetching lessons:", error);
            setLoading(false);
        });

        // Setup listener for chat messages
        const chatQuery = query(collection(db, chatCollectionPath));
        const unsubscribeChat = onSnapshot(chatQuery, (querySnapshot) => {
            const messagesData = [];
            querySnapshot.forEach((doc) => {
                messagesData.push({ id: doc.id, ...doc.data() });
            });
            // Sort messages by timestamp
            messagesData.sort((a, b) => (a.timestamp || 0) - (b.timestamp || 0));
            setMessages(messagesData);
        }, (error) => {
            console.error("Error fetching chat messages:", error);
        });

        // Cleanup listeners on component unmount
        return () => {
            unsubscribeLessons();
            unsubscribeChat();
        };
    }, [isAuthReady]);

    // Handle posting a new lesson
    const handlePostLesson = async (e) => {
        e.preventDefault();
        if (!user || lessonTitle.trim() === '' || lessonContent.trim() === '') {
            return;
        }

        const firebaseConfig = getFirebaseConfig();
        const app = initializeApp(firebaseConfig);
        const db = getFirestore(app);
        
        try {
            await addDoc(collection(db, `artifacts/${appId}/public/data/lessons`), {
                title: lessonTitle,
                content: lessonContent,
                authorId: user.uid,
                createdAt: Date.now(),
            });
            setLessonTitle('');
            setLessonContent('');
        } catch (e) {
            console.error("Error adding lesson document: ", e);
        }
    };

    // Handle sending a new chat message
    const handleSendMessage = async (e) => {
        e.preventDefault();
        if (!user || chatMessage.trim() === '') {
            return;
        }

        const firebaseConfig = getFirebaseConfig();
        const app = initializeApp(firebaseConfig);
        const db = getFirestore(app);

        try {
            await addDoc(collection(db, `artifacts/${appId}/public/data/chat`), {
                text: chatMessage,
                authorId: user.uid,
                timestamp: Date.now(),
            });
            setChatMessage('');
        } catch (e) {
            console.error("Error adding chat message: ", e);
        }
    };

    return (
        <div className="flex flex-col min-h-screen bg-gray-100 font-inter">
            {/* Header Section */}
            <header className="bg-white shadow-md py-4">
                <div className="container mx-auto px-4 md:px-8 flex flex-col md:flex-row justify-between items-center">
                    <div className="flex flex-col items-center md:items-start mb-4 md:mb-0">
                        <h1 className="text-2xl font-bold text-blue-600">Classroom Hub</h1>
                        {user && (
                            <p className="text-sm text-gray-500 mt-1 truncate w-full max-w-[200px]">User ID: {user.uid}</p>
                        )}
                    </div>
                    <nav className="flex space-x-4">
                        <a href="#lessons" className="text-gray-600 hover:text-blue-600 font-medium transition duration-300">Lessons</a>
                        <a href="#chat" className="text-gray-600 hover:text-blue-600 font-medium transition duration-300">Chat</a>
                    </nav>
                </div>
            </header>

            {/* Main Content */}
            <main className="flex-grow grid grid-cols-1 lg:grid-cols-3 gap-8 p-4 md:p-8">
                {/* Lessons and Notes Section */}
                <section id="lessons" className="lg:col-span-2 space-y-8">
                    <div className="bg-white rounded-xl shadow-lg p-6">
                        <h3 className="text-3xl font-bold text-center mb-6">Post a New Lesson or Note</h3>
                        <form onSubmit={handlePostLesson} className="space-y-4">
                            <input
                                type="text"
                                placeholder="Lesson Title"
                                value={lessonTitle}
                                onChange={(e) => setLessonTitle(e.target.value)}
                                className="w-full p-3 rounded-lg border-2 border-gray-300 focus:outline-none focus:ring-2 focus:ring-blue-500 transition duration-300"
                                required
                            />
                            <textarea
                                placeholder="Write your lesson or notes here..."
                                rows="8"
                                value={lessonContent}
                                onChange={(e) => setLessonContent(e.target.value)}
                                className="w-full p-3 rounded-lg border-2 border-gray-300 focus:outline-none focus:ring-2 focus:ring-blue-500 transition duration-300"
                                required
                            ></textarea>
                            <button
                                type="submit"
                                className="w-full bg-blue-600 text-white font-bold py-3 px-6 rounded-lg shadow-md hover:bg-blue-700 transition duration-300"
                            >
                                Post Lesson
                            </button>
                        </form>
                    </div>

                    <div className="bg-white rounded-xl shadow-lg p-6">
                        <h3 className="text-3xl font-bold text-center mb-6">All Lessons & Notes</h3>
                        {loading && (
                            <p className="text-center text-xl text-gray-500">Loading lessons...</p>
                        )}
                        {!loading && lessons.length === 0 && (
                            <p className="text-center text-xl text-gray-500">No lessons posted yet. Be the first to add one!</p>
                        )}
                        <div className="grid gap-6">
                            {lessons.map((lesson) => (
                                <div key={lesson.id} className="bg-gray-50 rounded-xl shadow-md p-6">
                                    <h4 className="text-xl font-semibold mb-2">{lesson.title}</h4>
                                    <p className="text-gray-700 mb-4">{lesson.content}</p>
                                    <p className="text-xs text-gray-400">
                                        Posted by: <span className="font-mono">{lesson.authorId}</span>
                                    </p>
                                </div>
                            ))}
                        </div>
                    </div>
                </section>

                {/* Chat Section */}
                <section id="chat" className="lg:col-span-1 flex flex-col space-y-6">
                    <div className="bg-white rounded-xl shadow-lg p-6 flex-1 flex flex-col">
                        <h3 className="text-3xl font-bold text-center mb-6">Student Chat</h3>
                        <div className="flex-1 overflow-y-auto space-y-4 p-2 mb-4 bg-gray-50 rounded-lg border border-gray-200">
                            {messages.length === 0 && (
                                <p className="text-center text-gray-500">Say hello!</p>
                            )}
                            {messages.map((msg) => (
                                <div key={msg.id} className={`flex ${msg.authorId === user?.uid ? 'justify-end' : 'justify-start'}`}>
                                    <div className={`p-3 rounded-xl max-w-[80%] shadow-sm ${msg.authorId === user?.uid ? 'bg-blue-200 rounded-br-none' : 'bg-gray-200 rounded-bl-none'}`}>
                                        <p className="font-semibold text-sm truncate max-w-[150px]">{msg.authorId}</p>
                                        <p className="text-gray-800 text-sm mt-1">{msg.text}</p>
                                    </div>
                                </div>
                            ))}
                        </div>
                        <form onSubmit={handleSendMessage} className="flex space-x-2">
                            <input
                                type="text"
                                placeholder="Type a message..."
                                value={chatMessage}
                                onChange={(e) => setChatMessage(e.target.value)}
                                className="flex-1 p-3 rounded-lg border-2 border-gray-300 focus:outline-none focus:ring-2 focus:ring-blue-500 transition duration-300"
                                required
                            />
                            <button
                                type="submit"
                                className="bg-blue-600 text-white font-bold py-3 px-4 rounded-lg shadow-md hover:bg-blue-700 transition duration-300 flex-shrink-0"
                            >
                                Send
                            </button>
                        </form>
                    </div>
                </section>
            </main>

            {/* Footer Section */}
            <footer className="bg-gray-800 text-white py-8 px-4 md:px-8">
                <div className="container mx-auto text-center">
                    <p>&copy; 2025 Classroom Hub. All rights reserved.</p>
                </div>
            </footer>
        </div>
    );
};

export default App;
