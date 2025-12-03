# Duos-Eats-New-demo
A discovery platform where users can avail discounts and offers
import React, { useState, useCallback, useEffect } from 'react';
import {
  Compass, Sparkles, X, MapPin, GalleryHorizontal, BookOpen,
  ChevronRight, ChevronLeft,
  // Icon imports for logos and UI
  ChefHat, Hotel, Dumbbell, Utensils, Palette, Database,
  Eye, Tag, BarChart3, Users, LayoutDashboard,
} from 'lucide-react';

// Firebase Imports
import { initializeApp } from 'firebase/app';
import {
  getAuth, signInAnonymously, signInWithCustomToken, onAuthStateChanged
} from 'firebase/auth';
import {
  getFirestore, doc, addDoc, setDoc, onSnapshot, collection, query, where, getDocs,
  updateDoc, increment
} from 'firebase/firestore';
import { setLogLevel, serverTimestamp } from 'firebase/firestore';


// --- Global Firebase and App Setup ---
const appId = typeof __app_id !== 'undefined' ? __app_id : 'default-app-id';
// NOTE: __firebase_config is a global variable provided by the environment.
const firebaseConfig = JSON.parse(typeof __firebase_config !== 'undefined' ? __firebase_config : '{}');

// Initialize Firebase services outside the component for single initialization
let app, db, auth;
let firebaseInitialized = false;

// Map to dynamically select the correct Lucide icon component
const iconMap = {
  ChefHat,
  Hotel,
  Dumbbell,
  Utensils,
  Palette,
  Default: Sparkles
};

// Mock Data for Initial Database Seeding (matches previous structure + logo and new stats)
const initialOffersSeed = [
  {
    type: 'Restaurant',
    name: 'The Golden Spoon',
    description: 'Enjoy 40% off your entire dining bill (excluding alcohol) on weekdays.',
    category: 'Dining',
    image: 'https://placehold.co/600x400/FF5722/ffffff?text=Restaurant+Offer',
    tag: 'Gourmet',
    location: 'Downtown City Center',
    address: '123 Main St, Cityville, CA 90210',
    photos: [
        'https://placehold.co/800x600/FF5722/ffffff?text=The+Golden+Spoon+Interior',
        'https://placehold.co/800x600/D32F2F/ffffff?text=Signature+Dish+1',
        'https://placehold.co/800x600/795548/ffffff?text=Outdoor+Seating',
    ],
    menu: [
        'https://placehold.co/800x1100/3E2723/ffffff?text=Dinner+Menu+Page+1',
        'https://placehold.co/800x1100/3E2723/ffffff?text=Wine+List',
    ],
    offersList: ['40% off Dining Bill (Weekdays)', 'Free Dessert with Main Course', '2-for-1 Appetizers on Tuesdays'],
    logoIcon: 'ChefHat',
    totalViews: 0, // New field
    totalClaims: 0, // New field
  },
  {
    type: 'Resort',
    name: 'Azure Waves Getaway',
    description: 'Book 3 nights, only pay for 2. Includes complimentary breakfast.',
    category: 'Travel',
    image: 'https://placehold.co/600x400/FF7043/ffffff?text=Resort+Deal',
    tag: 'Luxury Escape',
    location: 'Coastal Region',
    address: '456 Ocean View Blvd, Beach City, FL 33139',
    photos: [
        'https://placehold.co/800x600/0288D1/ffffff?text=Pool+and+Ocean',
        'https://placehold.co/800x600/00BCD4/ffffff?text=Resort+Room',
    ],
    menu: [
        'https://placehold.co/800x1100/1565C0/ffffff?text=Resort+Activity+Schedule',
    ],
    offersList: ['Book 3 nights, only pay for 2', '20% off Spa Services', 'Free Kids Club Access'],
    logoIcon: 'Hotel',
    totalViews: 0, // New field
    totalClaims: 0, // New field
  },
  {
    type: 'Lifestyle',
    name: 'Zen Body Fitness',
    description: 'First month free membership, plus 10% off all personal training packages.',
    category: 'Wellness',
    image: 'https://placehold.co/600x400/FF9800/ffffff?text=Gym+Pass',
    tag: 'Health',
    location: 'West Side Mall',
    address: '789 Mall Rd, Suite B, Center City, AZ 85001',
    photos: [
        'https://placehold.co/800x600/689F38/ffffff?text=Gym+Floor',
        'https://placehold.co/800x600/8BC34A/ffffff?text=Yoga+Studio',
    ],
    menu: [],
    offersList: ['First month free membership', '10% off Personal Training', 'Free Fitness Assessment'],
    logoIcon: 'Dumbbell',
    totalViews: 0, // New field
    totalClaims: 0, // New field
  },
];

// Firestore path constants
const PUBLIC_OFFERS_PATH = `artifacts/${appId}/public/data/offers`;

/**
 * Utility to populate the offers collection if it's empty.
 * Ensures new fields (views/claims) are present on first run.
 * @param {object} db - Firestore instance.
 */
const populateDatabase = async (db) => {
  try {
    if (!db) {
        console.warn("Database instance not available for population.");
        return false;
    }
    const q = query(collection(db, PUBLIC_OFFERS_PATH));
    const querySnapshot = await getDocs(q);

    if (querySnapshot.empty) {
      console.log("Database is empty. Populating with mock offers...");
      for (const offer of initialOffersSeed) {
        // Use addDoc to automatically generate a unique ID
        await addDoc(collection(db, PUBLIC_OFFERS_PATH), offer);
      }
      console.log("Mock offers added successfully.");
      return true; // Return success
    } else {
      console.log(`Offers collection found with ${querySnapshot.size} documents. Skipping population.`);
      // Optional: Check if existing documents need the new fields and update them
      querySnapshot.docs.forEach(async (docSnapshot) => {
        // Check for undefined to ensure new fields are present on existing docs
        if (docSnapshot.data().totalViews === undefined) {
          try {
            // Use updateDoc to add the fields without overwriting others
            await updateDoc(doc(db, PUBLIC_OFFERS_PATH, docSnapshot.id), {
              totalViews: docSnapshot.data().totalViews || 0,
              totalClaims: docSnapshot.data().totalClaims || 0,
            });
          } catch (e) {
             console.error(`Failed to update offer ${docSnapshot.id} with new fields:`, e);
          }
        }
      });
      return true; // Return success
    }
  } catch (error) {
    console.error("Error populating database:", error);
    return false;
  }
};


/**
 * Generates a unique, easy-to-read discount code.
 * @returns {string} The unique discount code.
 */
const generateUniqueCode = () => {
  // Use crypto.randomUUID for better uniqueness
  const uuid = crypto.randomUUID().substring(0, 8).toUpperCase();
  return `DISCOUNT-${uuid}`;
};

/**
 * Custom Modal Component for displaying the unique discount code.
 */
const DiscountModal = ({ offer, onClose, code, onCopy, copyStatus }) => {
  if (!offer) return null;

  return (
    <div className="fixed inset-0 bg-black bg-opacity-70 flex items-center justify-center p-4 z-50 transition-opacity duration-300">
      <div className="bg-white p-6 md:p-8 rounded-2xl shadow-2xl max-w-lg w-full transform scale-100 transition-transform duration-300">

        {/* Modal Header */}
        <div className="flex justify-between items-start border-b pb-4 mb-4">
          <h2 className="text-3xl font-extrabold text-orange-600 flex items-center">
            <Sparkles className="w-6 h-6 mr-2 fill-orange-400 text-orange-600"/>
            Offer Claimed
          </h2>
          {/* onClose is called here */}
          <button onClick={onClose} className="p-2 text-gray-500 hover:text-red-500 transition-colors rounded-full hover:bg-red-50">
            <X className="w-6 h-6" />
          </button>
        </div>

        {/* Offer Details */}
        <p className="text-xl font-semibold text-gray-800 mb-2">{offer.name}</p>
        <p className="text-sm text-gray-500 mb-6">{offer.description}</p>

        {/* Code Display */}
        <div className="text-center my-6 p-6 bg-orange-50 border-4 border-dashed border-orange-300 rounded-xl">
          <p className="text-lg text-gray-600 font-medium mb-3">Your Unique Discount Code:</p>
          <div className="flex items-center justify-center space-x-2">
            <code className="text-4xl font-mono font-bold text-orange-800 select-all tracking-wider break-all">
              {code}
            </code>
          </div>
        </div>

        {/* Instructions and Copy Button */}
        <div className="text-center">
          <p className="text-sm text-gray-600 mb-4">
            Show this code to the staff at **{offer.name}** to redeem your offer.
          </p>
          <button
            onClick={onCopy}
            className={`w-full py-3 px-6 text-white font-bold rounded-xl shadow-lg transition-all duration-200 transform ${
                copyStatus
                ? 'bg-green-600 hover:bg-green-700'
                : 'bg-orange-600 hover:bg-orange-700'
            } hover:scale-[1.02] active:scale-[0.98]`}
          >
            {copyStatus ? 'Code Copied! ✅' : 'Copy Code'}
          </button>
        </div>
      </div>
    </div>
  );
};

/**
 * Component for detailed brand information, photos, menu, and offers.
 */
const BrandDetailModal = ({ brand, onClose }) => {
  if (!brand) return null;

  // State to manage the currently visible photo/menu index
  const [photoIndex, setPhotoIndex] = useState(0);
  const [menuIndex, setMenuIndex] = useState(0);

  const totalPhotos = brand.photos ? brand.photos.length : 0;
  const totalMenus = brand.menu ? brand.menu.length : 0;

  const handleNextPhoto = () => setPhotoIndex((prev) => (prev + 1) % totalPhotos);
  const handlePrevPhoto = () => setPhotoIndex((prev) => (prev - 1 + totalPhotos) % totalPhotos);

  const handleNextMenu = () => setMenuIndex((prev) => (prev + 1) % totalMenus);
  const handlePrevMenu = () => setMenuIndex((prev) => (prev - 1 + totalMenus) % totalMenus);


  return (
    <div className="fixed inset-0 bg-black bg-opacity-70 flex items-center justify-center p-4 z-50 overflow-y-auto">
      <div className="bg-white p-4 md:p-8 rounded-2xl shadow-2xl max-w-4xl w-full my-8 transform transition-transform duration-300">

        {/* Modal Header */}
        <div className="flex justify-between items-start border-b pb-4 mb-6">
          <h2 className="text-4xl font-extrabold text-gray-800 flex items-center">
            {brand.name}
          </h2>
          {/* onClose is called here */}
          <button onClick={onClose} className="p-2 text-gray-500 hover:text-red-500 transition-colors rounded-full hover:bg-red-50">
            <X className="w-6 h-6" />
          </button>
        </div>

        {/* Address and Location */}
        <div className="flex items-center text-lg text-gray-600 mb-6 border-b pb-4">
            <MapPin className="w-5 h-5 mr-2 text-orange-500 flex-shrink-0" />
            <p className="font-medium">{brand.address}</p>
        </div>

        {/* Photo Gallery */}
        <section className="mb-8">
            <h3 className="text-2xl font-bold text-gray-800 mb-4 flex items-center">
                <GalleryHorizontal className="w-5 h-5 mr-2 text-orange-500" />
                Venue Photos
            </h3>
            {totalPhotos > 0 ? (
                <div className="relative bg-gray-100 rounded-xl overflow-hidden shadow-lg h-96">
                    <img
                        src={brand.photos[photoIndex]}
                        alt={`${brand.name} Photo ${photoIndex + 1}`}
                        className="w-full h-full object-cover"
                        onError={(e) => {
                            e.target.onerror = null;
                            e.target.src = 'https://placehold.co/800x600/94A3B8/ffffff?text=Image+Unavailable';
                        }}
                    />
                    {totalPhotos > 1 && (
                      <>
                        <button onClick={handlePrevPhoto} className="absolute left-4 top-1/2 -translate-y-1/2 p-3 bg-black/50 text-white rounded-full hover:bg-black/70 transition-colors">
                            <ChevronLeft className="w-6 h-6" />
                        </button>
                        <button onClick={handleNextPhoto} className="absolute right-4 top-1/2 -translate-y-1/2 p-3 bg-black/50 text-white rounded-full hover:bg-black/70 transition-colors">
                            <ChevronRight className="w-6 h-6" />
                        </button>
                        <div className="absolute bottom-4 left-0 right-0 text-center">
                            <span className="bg-black/60 text-white text-sm px-3 py-1 rounded-full">
                                {photoIndex + 1} / {totalPhotos}
                            </span>
                        </div>
                      </>
                    )}
                </div>
            ) : (
                <div className="text-center py-10 text-gray-500 bg-gray-100 rounded-xl">No photos available.</div>
            )}
        </section>

        {/* Menu Pictures */}
        {totalMenus > 0 && (
            <section className="mb-8">
                <h3 className="text-2xl font-bold text-gray-800 mb-4 flex items-center">
                    <BookOpen className="w-5 h-5 mr-2 text-orange-500" />
                    Menu Pictures
                </h3>
                <div className="relative bg-gray-100 rounded-xl overflow-hidden shadow-lg h-[600px] w-full max-w-md mx-auto">
                    <img
                        src={brand.menu[menuIndex]}
                        alt={`${brand.name} Menu ${menuIndex + 1}`}
                        className="w-full h-full object-contain"
                        onError={(e) => {
                            e.target.onerror = null;
                            e.target.src = 'https://placehold.co/800x1100/94A3B8/ffffff?text=Menu+Image+Unavailable';
                        }}
                    />
                    {totalMenus > 1 && (
                      <>
                        <button onClick={handlePrevMenu} className="absolute left-4 top-1/2 -translate-y-1/2 p-3 bg-black/50 text-white rounded-full hover:bg-black/70 transition-colors">
                            <ChevronLeft className="w-6 h-6" />
                        </button>
                        <button onClick={handleNextMenu} className="absolute right-4 top-1/2 -translate-y-1/2 p-3 bg-black/50 text-white rounded-full hover:bg-black/70 transition-colors">
                            <ChevronRight className="w-6 h-6" />
                        </button>
                        <div className="absolute bottom-4 left-0 right-0 text-center">
                            <span className="bg-black/60 text-white text-sm px-3 py-1 rounded-full">
                                Page {menuIndex + 1} / {totalMenus}
                            </span>
                        </div>
                      </>
                    )}
                </div>
            </section>
        )}

        {/* Current Offers List */}
        <section>
            <h3 className="text-2xl font-bold text-gray-800 mb-4 flex items-center">
                <Sparkles className="w-5 h-5 mr-2 text-orange-500" />
                All Current Offers
            </h3>
            <ul className="space-y-3">
                {brand.offersList && brand.offersList.map((offer, index) => (
                    <li key={index} className="flex items-start p-3 bg-orange-50 border-l-4 border-orange-500 rounded-lg shadow-sm">
                        <span className="text-orange-600 font-semibold mr-3">🎉</span>
                        <p className="text-gray-700">{offer}</p>
                    </li>
                ))}
            </ul>
        </section>

      </div>
    </div>
  );
};

// --- Partner Dashboard Component ---
const PartnerDashboard = ({ offers, onClose }) => {
    // Calculate total views and claims across all offers
    const totalViews = offers.reduce((sum, offer) => sum + (offer.totalViews || 0), 0);
    const totalClaims = offers.reduce((sum, offer) => sum + (offer.totalClaims || 0), 0);

    // Sort offers by claims descending for top performance
    const sortedOffers = [...offers].sort((a, b) => (b.totalClaims || 0) - (a.totalClaims || 0));

    return (
        <div className="fixed inset-0 bg-black bg-opacity-70 flex items-center justify-center p-4 z-50 overflow-y-auto">
            <div className="bg-white p-6 md:p-10 rounded-2xl shadow-2xl max-w-5xl w-full my-8 transform transition-transform duration-300">

                {/* Modal Header */}
                <div className="flex justify-between items-start border-b pb-4 mb-8">
                    <h2 className="text-3xl font-extrabold text-gray-800 flex items-center">
                        <LayoutDashboard className="w-7 h-7 mr-3 text-orange-600"/>
                        Partner Metrics Dashboard
                    </h2>
                    <button onClick={onClose} className="p-2 text-gray-500 hover:text-red-500 transition-colors rounded-full hover:bg-red-50">
                        <X className="w-6 h-6" />
                    </button>
                </div>

                {/* Summary Metrics */}
                <div className="grid grid-cols-1 sm:grid-cols-2 gap-6 mb-10">
                    <div className="bg-blue-50 p-6 rounded-xl border border-blue-200 flex items-center justify-between shadow-sm">
                        <div>
                            <p className="text-sm font-semibold text-blue-600 uppercase">Total Offer Views</p>
                            <p className="text-4xl font-extrabold text-gray-900 mt-1">{totalViews.toLocaleString()}</p>
                        </div>
                        <Eye className="w-10 h-10 text-blue-400 opacity-70" />
                    </div>
                    <div className="bg-green-50 p-6 rounded-xl border border-green-200 flex items-center justify-between shadow-sm">
                        <div>
                            <p className="text-sm font-semibold text-green-600 uppercase">Total Discounts Availed</p>
                            <p className="text-4xl font-extrabold text-gray-900 mt-1">{totalClaims.toLocaleString()}</p>
                        </div>
                        <Tag className="w-10 h-10 text-green-400 opacity-70" />
                    </div>
                </div>

                {/* Offer Performance Table */}
                <h3 className="text-2xl font-bold text-gray-800 mb-4 border-l-4 border-orange-500 pl-3 flex items-center">
                    <BarChart3 className="w-5 h-5 mr-2 text-orange-500" /> Offer Performance Breakdown
                </h3>

                <div className="overflow-x-auto bg-white rounded-xl shadow-lg border border-gray-100">
                    <table className="min-w-full divide-y divide-gray-200">
                        <thead className="bg-gray-50">
                            <tr>
                                <th className="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">Offer Name</th>
                                <th className="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">Category</th>
                                <th className="px-6 py-3 text-right text-xs font-medium text-gray-500 uppercase tracking-wider">Views</th>
                                <th className="px-6 py-3 text-right text-xs font-medium text-gray-500 uppercase tracking-wider">Claims</th>
                                <th className="px-6 py-3 text-right text-xs font-medium text-gray-500 uppercase tracking-wider">Claim Rate</th>
                            </tr>
                        </thead>
                        <tbody className="bg-white divide-y divide-gray-200">
                            {sortedOffers.map((offer) => {
                                const views = offer.totalViews || 0;
                                const claims = offer.totalClaims || 0;
                                // Calculate claim rate safely
                                const claimRate = views > 0 ? ((claims / views) * 100).toFixed(1) : 0;
                                return (
                                    <tr key={offer.id} className="hover:bg-orange-50 transition-colors">
                                        <td className="px-6 py-4 whitespace-nowrap text-sm font-medium text-gray-900">
                                            {offer.name}
                                        </td>
                                        <td className="px-6 py-4 whitespace-nowrap text-sm text-gray-500">
                                            {offer.type}
                                        </td>
                                        <td className="px-6 py-4 whitespace-nowrap text-sm text-right font-semibold text-blue-700">
                                            {views.toLocaleString()}
                                        </td>
                                        <td className="px-6 py-4 whitespace-nowrap text-sm text-right font-semibold text-green-700">
                                            {claims.toLocaleString()}
                                        </td>
                                        <td className="px-6 py-4 whitespace-nowrap text-sm text-right text-gray-700">
                                            <span className={`px-2 py-1 inline-flex text-xs leading-5 font-semibold rounded-full ${claimRate > 5 ? 'bg-green-100 text-green-800' : 'bg-yellow-100 text-yellow-800'}`}>
                                                {claimRate}%
                                            </span>
                                        </td>
                                    </tr>
                                );
                            })}
                        </tbody>
                    </table>
                </div>

            </div>
        </div>
    );
};

// --- Main App Component ---

const App = () => {
  // Application State
  const [offers, setOffers] = useState([]);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);
  const [userId, setUserId] = useState(null);
  const [isAuthReady, setIsAuthReady] = useState(false);
  const [dbPopulated, setDbPopulated] = useState(false);

  // Modal State
  const [discountModalOpen, setDiscountModalOpen] = useState(false);
  const [selectedOffer, setSelectedOffer] = useState(null);
  const [discountCode, setDiscountCode] = useState('');
  const [copyStatus, setCopyStatus] = useState(false);
  const [detailModalOpen, setDetailModalOpen] = useState(false);
  const [selectedBrandForDetails, setSelectedBrandForDetails] = useState(null);
  const [dashboardOpen, setDashboardOpen] = useState(false); // New state for Dashboard

  // --- Firebase Initialization and Auth ---
  useEffect(() => {
    try {
      if (!firebaseInitialized && firebaseConfig && Object.keys(firebaseConfig).length > 0) {
        setLogLevel('debug'); // Enable debug logging for Firestore
        app = initializeApp(firebaseConfig);
        db = getFirestore(app);
        auth = getAuth(app);
        firebaseInitialized = true;
      }

      const unsubscribeAuth = onAuthStateChanged(auth, async (user) => {
        if (user) {
          setUserId(user.uid);
          setIsAuthReady(true);
        } else {
          try {
            // Check for the initial custom auth token
            if (typeof __initial_auth_token !== 'undefined' && __initial_auth_token) {
              await signInWithCustomToken(auth, __initial_auth_token);
            } else {
              // Fallback to anonymous sign-in if no custom token is provided
              await signInAnonymously(auth);
            }
          } catch (e) {
            console.error("Auth sign-in failed:", e);
          } finally {
            setIsAuthReady(true);
          }
        }
      });

      return () => {
        if (unsubscribeAuth) unsubscribeAuth();
      };

    } catch (e) {
      console.error("Firebase Initialization Error:", e);
      setError("Failed to initialize the application.");
      setIsAuthReady(true);
    }
  }, []);

  // --- Data Population Trigger ---
  useEffect(() => {
    if (isAuthReady && !dbPopulated && db) {
        populateDatabase(db).then(success => {
            if (success) setDbPopulated(true);
        });
    }
  }, [isAuthReady, dbPopulated]);


  // --- Data Fetching (Offers) ---
  useEffect(() => {
    if (!isAuthReady || !db) return; // Wait for authentication/db initialization

    const offersCollectionRef = collection(db, PUBLIC_OFFERS_PATH);

    // Set up real-time listener for public offers
    const unsubscribeSnapshot = onSnapshot(offersCollectionRef, (snapshot) => {
      const fetchedOffers = snapshot.docs.map(doc => ({
        id: doc.id,
        ...doc.data()
      }));
      setOffers(fetchedOffers);
      setLoading(false);
    }, (err) => {
      console.error("Firestore error:", err);
      setError("Failed to load offers. Please check database permissions.");
      setLoading(false);
    });

    return () => unsubscribeSnapshot();
  }, [isAuthReady]);


  // --- Modal Close Handlers ---
  const closeDiscountModal = useCallback(() => {
    setDiscountModalOpen(false);
    setSelectedOffer(null);
    setDiscountCode('');
    setCopyStatus(false);
  }, []);

  const closeDetailModal = useCallback(() => {
    setDetailModalOpen(false);
    setSelectedBrandForDetails(null);
  }, []);

  const closeDashboard = useCallback(() => {
    setDashboardOpen(false);
  }, []);

  // --- Core Logic: Track Views ---
  const handleViewDetails = useCallback(async (offer) => {
    setSelectedBrandForDetails(offer);
    setDetailModalOpen(true);

    // Increment the totalViews counter in Firestore
    if (db && offer.id) {
        try {
            const offerRef = doc(db, PUBLIC_OFFERS_PATH, offer.id);
            // Use increment to update the counter atomically
            await updateDoc(offerRef, {
                totalViews: increment(1)
            });
        } catch (e) {
            console.error("Error incrementing view count:", e);
        }
    }
  }, []);

  // --- Core Logic: Claim Discount & Track Claims ---
  const handleAvailDiscount = useCallback(async (offer) => {
    if (!userId || !db) {
      console.error("Cannot claim discount: User or Database not ready.");
      return;
    }

    // Path for the user's private claimed codes
    const PRIVATE_CLAIMS_PATH = `artifacts/${appId}/users/${userId}/claimed_codes`;
    const claimsCollectionRef = collection(db, PRIVATE_CLAIMS_PATH);
    const offerRef = doc(db, PUBLIC_OFFERS_PATH, offer.id);

    // 1. Check if the code has already been claimed for this offer by this user
    const q = query(claimsCollectionRef, where('offerId', '==', offer.id));
    const snapshot = await getDocs(q);

    let codeToDisplay = '';

    if (!snapshot.empty) {
      // Code already claimed, retrieve existing code
      const existingClaim = snapshot.docs[0].data();
      codeToDisplay = existingClaim.code;
      console.log(`Offer already claimed. Using existing code: ${codeToDisplay}`);
    } else {
      // Code not claimed, generate and save new code
      const newCode = generateUniqueCode();
      codeToDisplay = newCode;

      try {
        // Save the claim to the user's private collection
        await addDoc(claimsCollectionRef, {
          offerId: offer.id,
          offerName: offer.name,
          code: newCode,
          claimedAt: serverTimestamp(),
        });
        console.log(`New code saved for offer ${offer.id}.`);

        // **NEW LOGIC:** Increment totalClaims only for NEW claims in the public offer doc
        await updateDoc(offerRef, {
            totalClaims: increment(1)
        });

      } catch (e) {
        console.error("Error saving new code/incrementing claims:", e);
        // Note: Even if Firestore fails, we show the code so the user isn't blocked.
        setError("Failed to save the claimed code to your profile, but here is your code.");
      }
    }

    // Open modal with the retrieved or new code
    setSelectedOffer(offer);
    setDiscountCode(codeToDisplay);
    setDiscountModalOpen(true);
    setCopyStatus(false);
  }, [userId]);

  // Function to handle code copying (uses document.execCommand('copy') for iFrame compatibility)
  const handleCopyCode = () => {
    const tempInput = document.createElement('textarea');
    tempInput.value = discountCode;
    document.body.appendChild(tempInput);
    tempInput.select();
    try {
      document.execCommand('copy');
      setCopyStatus(true);
      setTimeout(() => setCopyStatus(false), 2000);
    } catch (err) {
      console.error('Could not copy text: ', err);
    }
    document.body.removeChild(tempInput);
  };

  // Effect to close any modal using the ESC key
  useEffect(() => {
    const handleKeydown = (event) => {
      if (event.key === 'Escape') {
        if (discountModalOpen) closeDiscountModal();
        else if (detailModalOpen) closeDetailModal();
        else if (dashboardOpen) closeDashboard();
      }
    };

    document.addEventListener('keydown', handleKeydown);
    return () => {
      document.removeEventListener('keydown', handleKeydown);
    };
  }, [discountModalOpen, detailModalOpen, dashboardOpen, closeDiscountModal, closeDetailModal, closeDashboard]);


  // Offer Card Component
  const OfferCard = ({ offer, onViewDetails, onAvailDiscount }) => {
    let typeColor = '';

    const IconComponent = iconMap[offer.logoIcon] || iconMap.Default;

    switch (offer.type) {
      case 'Restaurant':
        typeColor = 'text-red-600 bg-red-100';
        break;
      case 'Resort':
        typeColor = 'text-blue-600 bg-blue-100';
        break;
      case 'Lifestyle':
        typeColor = 'text-green-600 bg-green-100';
        break;
      default:
        typeColor = 'text-gray-600 bg-gray-100';
    }

    return (
      <div className="bg-white rounded-xl shadow-xl overflow-hidden transition-all duration-300 border border-gray-100 group">

        {/* Card Image and Tag */}
        <div
          className="relative h-48 sm:h-56 bg-gray-200 cursor-pointer"
          onClick={() => onViewDetails(offer)} // Click image to view details
        >
          <img
            src={offer.image}
            alt={offer.name}
            className="w-full h-full object-cover transition-opacity duration-500 group-hover:opacity-90"
            onError={(e) => {
                e.target.onerror = null;
                e.target.src = `https://placehold.co/600x400/FF7043/ffffff?text=${offer.type}`;
            }}
          />
          <span className="absolute top-3 left-3 px-3 py-1 text-xs font-bold text-white bg-orange-600 rounded-full shadow-md">
            {offer.tag}
          </span>
          <div className="absolute inset-0 flex items-center justify-center bg-black bg-opacity-0 group-hover:bg-opacity-20 transition-all">
            <span className="text-white text-sm font-bold opacity-0 group-hover:opacity-100 transition-opacity p-2 bg-black/60 rounded-lg">View Details</span>
          </div>
        </div>

        {/* Card Content */}
        <div className="p-5 md:p-6">
          {/* Updated to use IconComponent as the logo */}
          <div className={`inline-flex items-center px-3 py-1 text-xs font-semibold rounded-full mb-3 ${typeColor}`}>
            <IconComponent className="w-4 h-4 mr-1 fill-current" /> {offer.type}
          </div>

          <h3
            className="text-2xl font-bold text-gray-800 mb-2 leading-snug cursor-pointer hover:text-orange-600 transition-colors"
            onClick={() => onViewDetails(offer)} // Click name to view details
          >
            {offer.name}
          </h3>

          <div className="flex items-center text-sm text-gray-500 mb-3">
            <MapPin className="w-4 h-4 mr-1 text-orange-500" />
            {offer.location}
          </div>

          <p className="text-gray-600 mb-6 line-clamp-3 min-h-[60px]">{offer.description}</p>

          <button
            onClick={() => onAvailDiscount(offer)}
            className="w-full py-3 px-4 bg-orange-500 text-white font-semibold rounded-xl shadow-lg shadow-orange-300/50 hover:bg-orange-600 transition-colors duration-200 active:bg-orange-700"
            disabled={!isAuthReady}
          >
            {isAuthReady ? 'Avail Discount' : 'Loading...'}
          </button>
        </div>
      </div>
    );
  };

  return (
    <div className="min-h-screen bg-gray-50 font-sans">

      {/* Header */}
      <header className="bg-orange-600 shadow-xl sticky top-0 z-40">
        <div className="max-w-7xl mx-auto p-4 sm:p-6 flex justify-between items-center">
          <div className="flex items-center">
            <Compass className="w-8 h-8 text-white mr-3" />
            <h1 className="text-3xl font-extrabold text-white tracking-tight">
              Duos Eats
            </h1>
          </div>
          <div className="flex items-center space-x-4">
             {/* Partner Dashboard Button */}
             <button
                onClick={() => setDashboardOpen(true)}
                className="flex items-center bg-white text-orange-600 hover:bg-orange-100 font-semibold py-2 px-4 rounded-xl shadow-md transition-colors"
             >
                <LayoutDashboard className="w-5 h-5 mr-2" />
                Partner Dashboard
             </button>

             {/* User ID Display */}
             <div className="text-white text-sm bg-orange-800/20 py-2 px-4 rounded-lg flex items-center">
                <Database className="w-4 h-4 mr-2" />
                User ID: <span className="font-mono text-xs ml-1 overflow-hidden whitespace-nowrap overflow-ellipsis">{userId || 'N/A'}</span>
            </div>
          </div>
        </div>
      </header>

      {/* Hero Section */}
      <main className="max-w-7xl mx-auto px-4 sm:px-6 py-12 md:py-16">
        <div className="text-center mb-12 md:mb-20 bg-orange-50 p-6 rounded-3xl border-4 border-orange-200">
          <h2 className="text-4xl sm:text-5xl font-extrabold text-gray-800 mb-4 leading-tight">
            Discover Your Next Adventure & Save Big
          </h2>
          <p className="text-xl text-orange-700 max-w-2xl mx-auto">
            Find the hottest discounts on the best restaurants, luxury resorts, and exciting lifestyle experiences near you.
          </p>
        </div>

        {/* Offer Grid Section */}
        <section id="offers" className="pb-12">
          <div className="flex justify-between items-center mb-8 flex-wrap">
            <h3 className="text-3xl font-bold text-gray-800 border-l-4 border-orange-500 pl-3 mb-4 sm:mb-0">
              Today's Featured Offers (Click Card for Details)
            </h3>
            <div className="flex space-x-4">
              {error && (
                  <div className="bg-red-500 text-white py-2 px-4 rounded-full text-sm font-semibold shadow-md transition-all duration-300 transform">
                      Error: {error}
                  </div>
              )}
              {loading && (
                  <div className="bg-blue-500 text-white py-2 px-4 rounded-full text-sm font-semibold shadow-md transition-all duration-300 transform">
                      Loading Offers...
                  </div>
              )}
            </div>
          </div>

          <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-8">
            {offers.length > 0 ? offers.map(offer => (
              <OfferCard
                key={offer.id}
                offer={offer}
                onViewDetails={handleViewDetails}
                onAvailDiscount={handleAvailDiscount}
              />
            )) : !loading && (
                <div className="md:col-span-2 lg:col-span-3 text-center py-20 text-gray-500 text-xl bg-white rounded-xl shadow-lg">
                    No offers found in the database.
                </div>
            )}
          </div>
        </section>
      </main>

      {/* Footer */}
      <footer className="bg-gray-800 text-white p-6 mt-12">
        <div className="max-w-7xl mx-auto text-center text-sm">
          &copy; {new Date().getFullYear()} Duos Eats. All rights reserved. | Simple, effective discounts.
        </div>
      </footer>

      {/* Modal Rendered Here: Discount Claim */}
      {discountModalOpen && (
        <DiscountModal
          offer={selectedOffer}
          onClose={closeDiscountModal}
          code={discountCode}
          onCopy={handleCopyCode}
          copyStatus={copyStatus}
        />
      )}

      {/* Modal Rendered Here: Brand Details */}
      {detailModalOpen && (
        <BrandDetailModal
          brand={selectedBrandForDetails}
          onClose={closeDetailModal}
        />
      )}

      {/* Modal Rendered Here: Partner Dashboard */}
      {dashboardOpen && (
        <PartnerDashboard
          offers={offers}
          onClose={closeDashboard}
        />
      )}
    </div>
  );
};

export default App;
