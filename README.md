# bwai-minna-demo
===========================================================================

# Building AIFeedback: A Next.js, Firebase, and Genkit Tutorial

This tutorial guides you through building the AIFeedback application, a platform where users can submit feedback, which is then analyzed for sentiment, keywords, and category using AI (Google's Gemini model via Genkit), and stored persistently in Firebase Firestore.

## Table of Contents

1.  [Introduction](#introduction)
2.  [Prerequisites](#prerequisites)
3.  [Project Setup](#project-setup)
    *   [Initialize Next.js App](#initialize-nextjs-app)
    *   [Install Dependencies](#install-dependencies)
    *   [Setup ShadCN/UI](#setup-shadcnui)
4.  [Firebase Setup](#firebase-setup)
    *   [Create Firebase Project](#create-firebase-project)
    *   [Configure Firestore](#configure-firestore)
    *   [Add Firebase Config to App](#add-firebase-config-to-app)
    *   [Initialize Firebase in Next.js](#initialize-firebase-in-nextjs)
5.  [Genkit (AI) Setup](#genkit-ai-setup)
    *   [Install Genkit CLI](#install-genkit-cli)
    *   [Initialize Genkit Configuration](#initialize-genkit-configuration)
    *   [Create Sentiment Analysis Flow](#create-sentiment-analysis-flow)
    *   [Genkit Development Server File](#genkit-development-server-file)
6.  [Backend Logic: Server Actions](#backend-logic-server-actions)
    *   [Define Data Types](#define-data-types)
    *   [Create Server Actions](#create-server-actions)
7.  [Building the User Interface](#building-the-user-interface)
    *   [Global Styles and Layout](#global-styles-and-layout)
    *   [Landing Page (`src/app/page.tsx`)](#landing-page-srcapppagetsx)
    *   [Feedback Page (`src/app/feedback/page.tsx`)](#feedback-page-srcappfeedbackpagetsx)
    *   [Feedback Form Component (`src/components/feedback-form.tsx`)](#feedback-form-component-srccomponentsfeedback-formtsx)
    *   [Comment List Component (`src/components/comment-list.tsx`)](#comment-list-component-srccomponentscomment-listtsx)
    *   [Sentiment Badge Component (`src/components/sentiment-badge.tsx`)](#sentiment-badge-component-srccomponentssentiment-badgetsx)
8.  [Running the Application](#running-the-application)
9.  [Environment Variables](#environment-variables)
10. [Conclusion](#conclusion)

## 1. Introduction

AIFeedback demonstrates how to integrate modern web technologies to create a smart feedback system.
Key features:
*   User-submitted feedback.
*   AI-powered sentiment analysis, keyword extraction, and category suggestion.
*   Persistent storage of feedback and analysis results.
*   A modern, responsive UI built with Next.js and ShadCN/UI.
*   Dark theme support for the landing page.

## 2. Prerequisites

*   **Node.js** (v18 or later recommended) and **npm** (or **yarn**).
*   A **Firebase account** and a new Firebase project.
*   A **Google Cloud Platform (GCP) project** with the Vertex AI API enabled.
*   **Google Cloud SDK** installed and authenticated (`gcloud auth application-default login`).
*   Familiarity with React, Next.js, and TypeScript.

## 3. Project Setup

### Initialize Next.js App

Start by creating a new Next.js application:

```bash
npx create-next-app@latest aifeedback-app --typescript --tailwind --eslint --app --src-dir --import-alias "@/*"
cd aifeedback-app
```

### Install Dependencies

Install the necessary packages:

```bash
npm install firebase genkit @genkit-ai/googleai @genkit-ai/next zod class-variance-authority clsx tailwind-merge tailwindcss-animate lucide-react @radix-ui/react-slot @hookform/resolvers react-hook-form date-fns dotenv patch-package uuid
npm install -D @types/uuid
```
And if you plan to use ShadCN UI components (which this project does heavily):
```bash
npm install @radix-ui/react-accordion @radix-ui/react-alert-dialog @radix-ui/react-avatar @radix-ui/react-checkbox @radix-ui/react-dialog @radix-ui/react-dropdown-menu @radix-ui/react-label @radix-ui/react-menubar @radix-ui/react-popover @radix-ui/react-progress @radix-ui/react-radio-group @radix-ui/react-scroll-area @radix-ui/react-select @radix-ui/react-separator @radix-ui/react-slider @radix-ui/react-switch @radix-ui/react-tabs @radix-ui/react-toast @radix-ui/react-tooltip recharts
```

### Setup ShadCN/UI

Initialize ShadCN/UI in your project. This will create a `components.json` file and setup necessary configurations.

```bash
npx shadcn-ui@latest init
```
You'll be prompted for configuration. For this project, the settings are typically:
*   **Style**: Default
*   **Base color**: Neutral (though we customize this later)
*   **CSS variables**: Yes
*   **`tailwind.config.js`**: `tailwind.config.ts`
*   **`globals.css`**: `src/app/globals.css`
*   **Components alias**: `@/components`
*   **Utils alias**: `@/lib/utils`
*   **React Server Components**: Yes

After initialization, you can add components like `button`, `card`, `textarea`, `toast`, etc.:
```bash
npx shadcn-ui@latest add button card textarea toast input label skeleton progress badge dialog dropdown-menu menubar popover scroll-area select separator slider switch tabs tooltip alert alert-dialog avatar checkbox form
```

## 4. Firebase Setup

### Create Firebase Project

1.  Go to the [Firebase Console](https://console.firebase.google.com/).
2.  Click "Add project" and follow the steps to create a new project.

### Configure Firestore

1.  In your Firebase project, go to "Firestore Database" in the left sidebar.
2.  Click "Create database".
3.  Start in **test mode** for development (you can change security rules later for production).
4.  Choose a Cloud Firestore location.

### Add Firebase Config to App

1.  In your Firebase project, go to "Project settings" (gear icon).
2.  Under "Your apps", click the web icon (`</>`) to add a web app.
3.  Register your app (give it a nickname, e.g., "AIFeedback Web").
4.  Firebase will provide you with a configuration object. Copy these SDK setup and configuration values.

Create a `.env.local` file in the root of your Next.js project and add your Firebase credentials:

```env
# .env.local

NEXT_PUBLIC_FIREBASE_API_KEY="YOUR_API_KEY"
NEXT_PUBLIC_FIREBASE_AUTH_DOMAIN="YOUR_AUTH_DOMAIN"
NEXT_PUBLIC_FIREBASE_PROJECT_ID="YOUR_PROJECT_ID"
NEXT_PUBLIC_FIREBASE_STORAGE_BUCKET="YOUR_STORAGE_BUCKET"
NEXT_PUBLIC_FIREBASE_MESSAGING_SENDER_ID="YOUR_MESSAGING_SENDER_ID"
NEXT_PUBLIC_FIREBASE_APP_ID="YOUR_APP_ID"

# You might also need Google Cloud Project ID for Genkit if it's different
# GOOGLE_CLOUD_PROJECT="YOUR_GCP_PROJECT_ID"
```
**Note:** Remember to add `.env.local` to your `.gitignore` file.

### Initialize Firebase in Next.js

Create `src/lib/firebase.ts` to initialize Firebase:

```typescript
// src/lib/firebase.ts
import { initializeApp, getApps, type FirebaseApp } from 'firebase/app';
import { getFirestore, type Firestore } from 'firebase/firestore';

const firebaseConfig = {
  apiKey: process.env.NEXT_PUBLIC_FIREBASE_API_KEY,
  authDomain: process.env.NEXT_PUBLIC_FIREBASE_AUTH_DOMAIN,
  projectId: process.env.NEXT_PUBLIC_FIREBASE_PROJECT_ID,
  storageBucket: process.env.NEXT_PUBLIC_FIREBASE_STORAGE_BUCKET,
  messagingSenderId: process.env.NEXT_PUBLIC_FIREBASE_MESSAGING_SENDER_ID,
  appId: process.env.NEXT_PUBLIC_FIREBASE_APP_ID,
};

let app: FirebaseApp;
let db: Firestore;

// Ensure Firebase is initialized only once
if (typeof window !== 'undefined' && !getApps().length) {
  app = initializeApp(firebaseConfig);
  db = getFirestore(app);
} else if (getApps().length > 0) { // Handle cases where it might have been initialized (e.g. HMR)
  app = getApps()[0];
  db = getFirestore(app);
} else { // Server-side initialization (e.g., for Server Actions)
  app = initializeApp(firebaseConfig);
  db = getFirestore(app);
}

export { app, db };
```

## 5. Genkit (AI) Setup

Genkit is used to interact with Google's Gemini AI model.

### Install Genkit CLI
If you haven't already, install the Genkit CLI globally or as a dev dependency:
```bash
npm install -g genkit-cli # or npm install -D genkit-cli
```

### Initialize Genkit Configuration

Create `src/ai/genkit.ts`:

```typescript
// src/ai/genkit.ts
import {genkit} from 'genkit';
import {googleAI} from '@genkit-ai/googleai';

export const ai = genkit({
  plugins: [googleAI()], // Ensure your GOOGLE_CLOUD_PROJECT env var is set, or configure explicitly
  model: 'googleai/gemini-2.0-flash', // Default model
});
```
Make sure your Google Cloud project is configured for API access and billing, and you've authenticated with `gcloud auth application-default login`.

### Create Sentiment Analysis Flow

Create `src/ai/flows/analyze-sentiment.ts`:

```typescript
// src/ai/flows/analyze-sentiment.ts
'use server';

/**
 * @fileOverview Sentiment analysis AI agent.
 *
 * - analyzeSentiment - A function that analyzes the sentiment of a given text.
 * - AnalyzeSentimentInput - The input type for the analyzeSentiment function.
 * - AnalyzeSentimentOutput - The return type for the analyzeSentiment function.
 */

import {ai} from '@/ai/genkit';
import {z} from 'genkit';

const AnalyzeSentimentInputSchema = z.object({
  text: z.string().describe('The text to analyze for sentiment.'),
});
export type AnalyzeSentimentInput = z.infer<typeof AnalyzeSentimentInputSchema>;

const AnalyzeSentimentOutputSchema = z.object({
  sentiment: z
    .enum(['positive', 'negative', 'neutral'])
    .describe('The sentiment of the text.'),
  emoji: z.string().describe('An emoji representing the sentiment.'),
  confidenceScore: z.number().min(0).max(1).describe('Confidence score (0-1) of the sentiment analysis.'),
  keywords: z.array(z.string()).max(5).describe('Up to 5 keywords from the text that most influenced the sentiment.'),
  suggestedCategory: z.string().describe('A suggested category for the feedback (e.g., Speaker, Venue, Food, Topic, Overall Experience, Networking, Workshop).'),
});
export type AnalyzeSentimentOutput = z.infer<typeof AnalyzeSentimentOutputSchema>;

// Mapping for consistent emoji representation
const sentimentToEmojiMap: Record<AnalyzeSentimentOutput['sentiment'], string> = {
  positive: '😄',
  negative: '😔',
  neutral: '😐',
};

// Exported wrapper function to call the flow
export async function analyzeSentiment(input: AnalyzeSentimentInput): Promise<AnalyzeSentimentOutput> {
  return analyzeSentimentFlow(input);
}

// Define the Genkit Prompt
const prompt = ai.definePrompt({
  name: 'analyzeSentimentPrompt',
  input: {schema: AnalyzeSentimentInputSchema},
  output: {schema: AnalyzeSentimentOutputSchema},
  prompt: `Analyze the sentiment of the following text. 
Respond with:
1. The sentiment (positive, negative, or neutral).
2. An emoji representing the sentiment (based on the sentiment, pick one from: 😄 for positive, 😔 for negative, 😐 for neutral).
3. A confidence score (a number between 0.0 and 1.0) for your sentiment analysis.
4. A list of up to 5 keywords from the text that most influenced the sentiment.
5. A suggested category for this feedback (e.g., Speaker, Venue, Food, Topic, Overall Experience, Networking, Workshop).

Text: {{{text}}}

Respond in JSON format.`,
});

// Define the Genkit Flow
const analyzeSentimentFlow = ai.defineFlow(
  {
    name: 'analyzeSentimentFlow',
    inputSchema: AnalyzeSentimentInputSchema,
    outputSchema: AnalyzeSentimentOutputSchema,
  },
  async input => {
    const {output} = await prompt(input); // Call the prompt
    if (!output) {
      throw new Error('AI analysis failed to produce an output.');
    }
    // Ensure emoji consistency and provide defaults if AI misses something
    const finalEmoji = sentimentToEmojiMap[output.sentiment] || '🤔'; // Fallback emoji
    
    return {
      sentiment: output.sentiment,
      emoji: finalEmoji,
      confidenceScore: output.confidenceScore !== undefined ? output.confidenceScore : 0.5, // Default confidence
      keywords: output.keywords || [],
      suggestedCategory: output.suggestedCategory || 'General', // Default category
    };
  }
);
```

### Genkit Development Server File

Create `src/ai/dev.ts` to run your Genkit flows locally during development:

```typescript
// src/ai/dev.ts
import { config } from 'dotenv';
config(); // Load .env variables

// Import your flows here to make them available to the Genkit dev server
import '@/ai/flows/analyze-sentiment.ts';

// You can add more flow imports as you create them
```

## 6. Backend Logic: Server Actions

We'll use Next.js Server Actions to handle form submissions and data fetching.

### Define Data Types

Create `src/types/index.ts` for shared TypeScript types:

```typescript
// src/types/index.ts
export type CommentSentiment = 'positive' | 'negative' | 'neutral';

export interface Comment {
  id: string;
  text: string;
  timestamp: Date; // Will be a Date object on the client
  sentiment: CommentSentiment;
  emoji: string;
  confidenceScore?: number;
  keywords?: string[];
  suggestedCategory?: string;
}
```

### Create Server Actions

Create `src/app/actions.ts`:

```typescript
// src/app/actions.ts
'use server';

import { analyzeSentiment, type AnalyzeSentimentInput, type AnalyzeSentimentOutput } from '@/ai/flows/analyze-sentiment';
import type { CommentSentiment } from '@/types'; // Using Comment from types isn't ideal here due to Date vs Timestamp
import { db } from '@/lib/firebase';
import { collection, addDoc, getDoc, serverTimestamp, query, orderBy, getDocs, limit, Timestamp, doc } from 'firebase/firestore';

// This type represents the data structure as it's returned by server actions,
// with timestamp as an ISO string for client-side hydration.
export interface RawCommentData {
  id: string;
  text: string;
  timestamp: string; // ISO string
  sentiment: CommentSentiment;
  emoji: string;
  confidenceScore?: number;
  keywords?: string[];
  suggestedCategory?: string;
}

export async function submitFeedbackAction(text: string): Promise<RawCommentData> {
  // Basic validation
  if (!text.trim()) throw new Error('Comment cannot be empty.');
  if (text.trim().length < 10) throw new Error('Comment must be at least 10 characters long.');
  if (text.trim().length > 500) throw new Error('Comment must not exceed 500 characters.');

  const analysisInput: AnalyzeSentimentInput = { text };
  const analysisOutput: AnalyzeSentimentOutput = await analyzeSentiment(analysisInput);

  const commentDataToSave = {
    text,
    sentiment: analysisOutput.sentiment,
    emoji: analysisOutput.emoji,
    confidenceScore: analysisOutput.confidenceScore,
    keywords: analysisOutput.keywords,
    suggestedCategory: analysisOutput.suggestedCategory,
    timestamp: serverTimestamp(), // Use Firestore server timestamp
  };

  try {
    const docRef = await addDoc(collection(db, 'feedbackComments'), commentDataToSave);
    // It's good practice to fetch the document to get the server-generated timestamp
    const newDocSnapshot = await getDoc(doc(db, 'feedbackComments', docRef.id));
    
    if (!newDocSnapshot.exists()) {
        throw new Error('Failed to retrieve newly created comment from database.');
    }

    const savedData = newDocSnapshot.data();
    const firestoreTimestamp = savedData.timestamp as Timestamp; // Cast to Firestore Timestamp

    // Prepare data for client, converting Timestamp to ISO string
    const newComment: RawCommentData = {
      id: newDocSnapshot.id,
      text: savedData.text,
      sentiment: savedData.sentiment,
      emoji: savedData.emoji,
      confidenceScore: savedData.confidenceScore,
      keywords: savedData.keywords,
      suggestedCategory: savedData.suggestedCategory,
      timestamp: firestoreTimestamp.toDate().toISOString(),
    };
    return newComment;

  } catch (error) {
    console.error('Error saving comment to Firestore:', error);
    if (error instanceof Error) {
      throw new Error(`Failed to save feedback: ${error.message}`);
    }
    throw new Error('Failed to save feedback due to an unknown error.');
  }
}

export async function getFeedbackCommentsAction(count: number = 20): Promise<RawCommentData[]> {
  try {
    const commentsRef = collection(db, 'feedbackComments');
    const q = query(commentsRef, orderBy('timestamp', 'desc'), limit(count));
    const querySnapshot = await getDocs(q);

    const comments: RawCommentData[] = [];
    querySnapshot.forEach((doc) => {
      const data = doc.data();
      const firestoreTimestamp = data.timestamp as Timestamp;
      comments.push({
        id: doc.id,
        text: data.text,
        timestamp: firestoreTimestamp.toDate().toISOString(),
        sentiment: data.sentiment,
        emoji: data.emoji,
        confidenceScore: data.confidenceScore,
        keywords: data.keywords,
        suggestedCategory: data.suggestedCategory,
      });
    });
    return comments;
  } catch (error) {
    console.error('Error fetching comments from Firestore:', error);
    return []; // Return empty array on error for client
  }
}
```

## 7. Building the User Interface

### Global Styles and Layout

1.  **Tailwind Configuration (`tailwind.config.ts`):**
    Update your `tailwind.config.ts` to include ShadCN colors and Geist font. The project files already contain a complete example. Key parts:
    *   Import `fontFamily` from `tailwindcss/defaultTheme`.
    *   Add `fontFamily.sans` and `fontFamily.mono` using CSS variables for Geist.
    *   Define color palettes for `background`, `foreground`, `primary`, `accent`, etc., using HSL CSS variables.
    *   Include `tailwindcss-animate` plugin.

2.  **Global CSS (`src/app/globals.css`):**
    This file sets up Tailwind base layers and defines CSS variables for the light and dark themes. Refer to the provided `src/app/globals.css` for the full theme setup. It uses HSL values for colors like `--background`, `--foreground`, `--primary`, `--accent`.

3.  **Root Layout (`src/app/layout.tsx`):**
    Sets up the HTML structure, includes global fonts (Geist Sans and Mono), and renders the `Toaster` component for notifications.

    ```typescript
    // src/app/layout.tsx
    import type { Metadata } from 'next';
    import { Geist } from 'next/font/google'; // Using Geist directly instead of Geist_Sans
    import { Geist_Mono } from 'next/font/google';
    import './globals.css';
    import { Toaster } from "@/components/ui/toaster"; // For ShadCN toasts

    const geistSans = Geist({ // Updated font import
      variable: '--font-geist-sans',
      subsets: ['latin'],
      display: 'swap',
    });

    const geistMono = Geist_Mono({
      variable: '--font-geist-mono',
      subsets: ['latin'],
      display: 'swap',
    });

    export const metadata: Metadata = {
      title: 'AIFeedback - Your Event, Understood',
      description: 'AI-powered feedback analysis for events.',
    };

    export default function RootLayout({
      children,
    }: Readonly<{
      children: React.ReactNode;
    }>) {
      return (
        <html lang="en" className="h-full">
          <body 
            className={`${geistSans.variable} ${geistMono.variable} font-sans antialiased flex flex-col min-h-screen`}
          >
            <div className="flex-grow">
              {children}
            </div>
            <Toaster />
          </body>
        </html>
      );
    }
    ```

### Landing Page (`src/app/page.tsx`)

This page serves as the entry point, showcasing the app's features with a dark theme.
It uses `Link` for navigation, `Button`, `Card` from ShadCN, and `lucide-react` icons.
The structure typically includes:
*   Header with logo and navigation.
*   Hero section with a call to action.
*   Features section highlighting benefits.
*   Another call to action section.
*   Footer.

Refer to the provided `src/app/page.tsx` for the full implementation. It heavily utilizes Tailwind CSS for styling and layout.

### Feedback Page (`src/app/feedback/page.tsx`)

This is the core interactive page where users submit and view feedback.

```typescript
// src/app/feedback/page.tsx
'use client';

import { useState, useEffect } from 'react';
import type { Comment } from '@/types';
import FeedbackForm from '@/components/feedback-form';
import CommentList from '@/components/comment-list';
import { submitFeedbackAction, getFeedbackCommentsAction, type RawCommentData } from '../actions';
import { useToast } from "@/hooks/use-toast"; // From ShadCN setup
import Image from 'next/image';
import { Skeleton } from "@/components/ui/skeleton";
import Link from 'next/link';
import { Button } from '@/components/ui/button';
import { Home } from 'lucide-react';

// Helper to convert raw comment data (with ISO string timestamp) to client-side Comment type (with Date object)
const mapRawCommentToComment = (raw: RawCommentData): Comment => ({
  ...raw,
  timestamp: new Date(raw.timestamp),
});

export default function FeedbackPage() {
  const [comments, setComments] = useState<Comment[]>([]);
  const [isLoading, setIsLoading] = useState(false); // For form submission
  const [isFetchingComments, setIsFetchingComments] = useState(true); // For initial comment load
  const { toast } = useToast();

  useEffect(() => {
    const fetchComments = async () => {
      setIsFetchingComments(true);
      try {
        const rawComments = await getFeedbackCommentsAction(50); // Fetch initial 50
        setComments(rawComments.map(mapRawCommentToComment));
      } catch (error) {
        toast({
          title: "Error Loading Comments",
          description: "Could not load previous feedback.",
          variant: "destructive",
        });
      } finally {
        setIsFetchingComments(false);
      }
    };
    fetchComments();
  }, [toast]);

  const handleAddComment = async (text: string) => {
    setIsLoading(true);
    try {
      const newRawCommentData = await submitFeedbackAction(text);
      const newComment = mapRawCommentToComment(newRawCommentData);
      setComments(prevComments => [newComment, ...prevComments]); // Add new comment to the top
      toast({
        title: "Feedback Submitted! 🎉",
        description: "Thanks for sharing your thoughts.",
        variant: "default",
      });
    } catch (error) {
      toast({
        title: "Submission Error",
        description: error instanceof Error ? error.message : "Could not submit your feedback.",
        variant: "destructive",
      });
    } finally {
      setIsLoading(false);
    }
  };

  return (
    <div className="container mx-auto max-w-2xl py-8 px-4 sm:py-12">
      <div className="mb-6">
        <Link href="/" passHref>
          <Button variant="outline" size="sm">
            <Home className="mr-2 h-4 w-4" />
            Back to Home
          </Button>
        </Link>
      </div>
      <header className="mb-8 sm:mb-10 text-center">
        <Image 
            src="https://picsum.photos/seed/eventlogo/150/50" // Placeholder logo
            alt="Event Logo" 
            width={120} 
            height={40} 
            className="mx-auto mb-4 rounded"
            data-ai-hint="event logo"
            priority
        />
        <h1 className="text-3xl sm:text-4xl font-bold text-primary tracking-tight">
          AIFeedback Platform
        </h1>
        <p className="mt-2 text-sm sm:text-base text-muted-foreground">
          Share your experience and see AI analyze it!
        </p>
      </header>
      
      <main>
        <FeedbackForm onSubmit={handleAddComment} isLoading={isLoading} />
        
        <section className="mt-10 sm:mt-14">
          <h2 className="text-xl sm:text-2xl font-semibold mb-5 sm:mb-6 text-center sm:text-left text-foreground">
            Participant Comments
          </h2>
          {isFetchingComments ? (
            <div className="space-y-4">
              <Skeleton className="h-24 w-full rounded-lg" />
              <Skeleton className="h-24 w-full rounded-lg" />
            </div>
          ) : (
            <CommentList comments={comments} />
          )}
        </section>
      </main>

      <footer className="mt-12 sm:mt-16 pt-8 border-t text-center">
        <p className="text-xs text-muted-foreground">
          &copy; {new Date().getFullYear()} AIFeedback. Powered by Genkit.
        </p>
      </footer>
    </div>
  );
}
```

### Feedback Form Component (`src/components/feedback-form.tsx`)

Uses `react-hook-form` with `zod` for validation.

```typescript
// src/components/feedback-form.tsx
'use client';

import * as React from 'react'; // Ensure React is imported
import { useForm, type SubmitHandler } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import * as z from 'zod';
import { Button } from '@/components/ui/button';
import { Textarea } from '@/components/ui/textarea';
import { Form, FormControl, FormField, FormItem, FormLabel, FormMessage } from '@/components/ui/form';
import { Loader2 } from 'lucide-react';

const feedbackSchema = z.object({
  comment: z.string()
    .min(10, { message: "Comment must be at least 10 characters long." })
    .max(500, { message: "Comment must not exceed 500 characters." }),
});

type FeedbackFormData = z.infer<typeof feedbackSchema>;

interface FeedbackFormProps {
  onSubmit: (comment: string) => Promise<void>;
  isLoading: boolean;
}

export default function FeedbackForm({ onSubmit, isLoading }: FeedbackFormProps) {
  const form = useForm<FeedbackFormData>({
    resolver: zodResolver(feedbackSchema),
    defaultValues: {
      comment: '',
    },
  });

  const handleFormSubmit: SubmitHandler<FeedbackFormData> = async (data) => {
    await onSubmit(data.comment);
    // Only reset if not loading (meaning submission likely succeeded or parent handled error)
    // The `useEffect` below provides a more robust reset mechanism.
  };
  
  // Effect to reset form when isLoading becomes false AFTER a successful submission attempt
  React.useEffect(() => {
    if (!isLoading && form.formState.isSubmitSuccessful) {
      form.reset();
    }
  }, [isLoading, form, form.formState.isSubmitSuccessful]);

  return (
    <Form {...form}>
      <form onSubmit={form.handleSubmit(handleFormSubmit)} className="space-y-6 bg-card p-6 rounded-lg shadow-md border">
        <FormField
          control={form.control}
          name="comment"
          render={({ field }) => (
            <FormItem>
              <FormLabel htmlFor="comment" className="text-lg font-medium text-foreground">Your Feedback</FormLabel>
              <FormControl>
                <Textarea
                  id="comment"
                  placeholder="Tell us what you think..."
                  className="min-h-[120px] resize-none bg-background focus:ring-accent"
                  {...field}
                  disabled={isLoading}
                />
              </FormControl>
              <FormMessage />
            </FormItem>
          )}
        />
        <Button type="submit" className="w-full bg-accent hover:bg-accent/90 text-accent-foreground focus-visible:ring-accent" disabled={isLoading}>
          {isLoading ? <Loader2 className="mr-2 h-4 w-4 animate-spin" /> : null}
          {isLoading ? 'Submitting...' : 'Submit Feedback'}
        </Button>
      </form>
    </Form>
  );
}
```

### Comment List Component (`src/components/comment-list.tsx`)

Displays a list of comments using `Card` and other UI elements.

```typescript
// src/components/comment-list.tsx
'use client';

import type { Comment } from '@/types';
import { Card, CardContent, CardDescription, CardFooter, CardHeader } from '@/components/ui/card';
import { Badge } from '@/components/ui/badge';
import { Clock, Tag, Percent, Sparkles } from 'lucide-react';
import { formatDistanceToNow } from 'date-fns';
import { SentimentBadge } from '@/components/sentiment-badge';
import { Progress } from '@/components/ui/progress';

interface CommentListProps {
  comments: Comment[];
}

export default function CommentList({ comments }: CommentListProps) {
  if (comments.length === 0) {
    return (
      <div className="text-center py-10">
        <p className="text-xl text-muted-foreground">No feedback yet.</p>
        <p className="text-muted-foreground">Be the first to share your thoughts!</p>
      </div>
    );
  }

  return (
    <div className="space-y-6">
      {comments.map((comment) => (
        <Card key={comment.id} className="shadow-lg bg-card border rounded-lg overflow-hidden flex flex-col">
          <CardHeader className="pb-3 pt-4 px-5 bg-card">
            <div className="flex justify-between items-start">
              <SentimentBadge sentiment={comment.sentiment} emoji={comment.emoji} />
              <CardDescription className="flex items-center gap-1.5 text-xs text-muted-foreground pt-1">
                <Clock className="h-3.5 w-3.5" />
                {comment.timestamp ? formatDistanceToNow(comment.timestamp, { addSuffix: true }) : 'Just now'}
              </CardDescription>
            </div>
          </CardHeader>
          <CardContent className="px-5 py-4 flex-grow">
            <p className="text-foreground leading-relaxed whitespace-pre-wrap">{comment.text}</p>
          </CardContent>
          {((comment.keywords && comment.keywords.length > 0) || comment.suggestedCategory || comment.confidenceScore !== undefined) && (
            <CardFooter className="px-5 py-3 bg-muted/30 border-t flex flex-col items-start space-y-2">
              {comment.suggestedCategory && (
                <div className="flex items-center space-x-2 text-xs text-muted-foreground">
                  <Tag className="h-3.5 w-3.5 text-primary" />
                  <span>Category: <Badge variant="secondary" className="font-normal">{comment.suggestedCategory}</Badge></span>
                </div>
              )}
              {comment.keywords && comment.keywords.length > 0 && (
                <div className="flex items-center space-x-1 text-xs text-muted-foreground flex-wrap">
                  <Sparkles className="h-3.5 w-3.5 text-primary mr-1" />
                  <span>Keywords:</span>
                  {comment.keywords.map((keyword, index) => (
                    <Badge key={index} variant="outline" className="font-normal bg-background">{keyword}</Badge>
                  ))}
                </div>
              )}
              {comment.confidenceScore !== undefined && (
                <div className="w-full text-xs text-muted-foreground">
                  <div className="flex items-center space-x-2 mb-0.5">
                     <Percent className="h-3.5 w-3.5 text-primary" />
                     <span>AI Confidence: {(comment.confidenceScore * 100).toFixed(0)}%</span>
                  </div>
                  <Progress value={comment.confidenceScore * 100} className="h-1.5" />
                </div>
              )}
            </CardFooter>
          )}
        </Card>
      ))}
    </div>
  );
}
```

### Sentiment Badge Component (`src/components/sentiment-badge.tsx`)

A small component to visually represent sentiment.

```typescript
// src/components/sentiment-badge.tsx
'use client';

import type { CommentSentiment } from '@/types';
import { Badge } from '@/components/ui/badge';
import { Smile, Frown, Meh } from 'lucide-react';
import { cn } from '@/lib/utils'; // from ShadCN setup

interface SentimentBadgeProps {
  sentiment: CommentSentiment;
  emoji: string;
  className?: string;
}

const sentimentStyles: Record<CommentSentiment, {
  icon: React.ElementType;
  bgColor: string;
  textColor: string;
  borderColor: string;
  emojiBg?: string;
}> = {
  positive: { icon: Smile, bgColor: 'bg-green-100 dark:bg-green-800/30', textColor: 'text-green-700 dark:text-green-400', borderColor: 'border-green-300 dark:border-green-700', emojiBg: 'bg-green-500/10' },
  negative: { icon: Frown, bgColor: 'bg-red-100 dark:bg-red-800/30', textColor: 'text-red-700 dark:text-red-400', borderColor: 'border-red-300 dark:border-red-700', emojiBg: 'bg-red-500/10' },
  neutral: { icon: Meh, bgColor: 'bg-yellow-100 dark:bg-yellow-800/30', textColor: 'text-yellow-700 dark:text-yellow-400', borderColor: 'border-yellow-300 dark:border-yellow-700', emojiBg: 'bg-yellow-500/10' },
};

export function SentimentBadge({ sentiment, emoji, className }: SentimentBadgeProps) {
  const styles = sentimentStyles[sentiment];
  if (!styles) return null; // Handle unknown sentiment if necessary
  const IconComponent = styles.icon;

  return (
    <div className={cn("flex items-center space-x-2", className)}>
      <Badge
        variant="outline"
        className={cn(
          "py-1 px-2 text-xs font-medium flex items-center gap-1.5 border rounded-md",
          styles.bgColor,
          styles.textColor,
          styles.borderColor
        )}
      >
        <IconComponent className={cn("h-3.5 w-3.5", styles.textColor)} />
        <span>{sentiment.charAt(0).toUpperCase() + sentiment.slice(1)}</span>
      </Badge>
      <span className={cn("text-lg p-0.5 rounded-md", styles.emojiBg)}>{emoji}</span>
    </div>
  );
}
```

## 8. Running the Application

You need to run two processes concurrently:
1.  The Next.js development server.
2.  The Genkit development server for your AI flows.

Open two terminal windows:

**Terminal 1 (Next.js App):**
```bash
npm run dev
```
This usually starts your app on `http://localhost:3000` (or `http://localhost:9002` as per `package.json`).

**Terminal 2 (Genkit Flows):**
```bash
npm run genkit:dev # or genkit start -- tsx src/ai/dev.ts
```
This starts the Genkit development UI, usually on `http://localhost:4000`, where you can inspect and test your flows.

## 9. Environment Variables

Ensure your `.env.local` file is correctly set up with Firebase and potentially Google Cloud credentials:

```env
# .env.local - Example
NEXT_PUBLIC_FIREBASE_API_KEY="YOUR_API_KEY"
NEXT_PUBLIC_FIREBASE_AUTH_DOMAIN="YOUR_AUTH_DOMAIN"
NEXT_PUBLIC_FIREBASE_PROJECT_ID="YOUR_PROJECT_ID"
# ... other Firebase variables

# For Genkit, if not implicitly picked up by gcloud ADC:
# GOOGLE_CLOUD_PROJECT="YOUR_GOOGLE_CLOUD_PROJECT_ID_FOR_AI_MODELS"
```
You might also need to set `GOOGLE_APPLICATION_CREDENTIALS` if you're using a service account JSON file for authentication with Google Cloud services, though `gcloud auth application-default login` is often preferred for local development.

## 10. Conclusion

You've now built a functional AIFeedback application! This project demonstrates:
*   Setting up a Next.js project with TypeScript and Tailwind CSS.
*   Integrating Firebase Firestore for data persistence.
*   Using Genkit to create AI flows with Google's Gemini model for sentiment analysis.
*   Building a user-friendly interface with ShadCN/UI components.
*   Implementing server actions for backend logic.

From here, you can expand the application by:
*   Adding user authentication.
*   Implementing more sophisticated data visualization for feedback trends.
*   Adding more AI features (e.g., summarization, translation).
*   Writing more robust error handling and security rules for Firestore.
*   Deploying the application to a platform like Vercel or Firebase Hosting.

Happy coding!
