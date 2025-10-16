# Zapra.ai - MVP Implementation Plan

## Table of Contents
1. [Project Overview](#project-overview)
2. [Architecture](#architecture)
3. [Tech Stack](#tech-stack)
4. [Implementation Phases](#implementation-phases)
5. [Appendix](#appendix)

---

## Project Overview

**Goal:** Build an MVP of Zapra.ai - a web application that allows users to record screen demos with voice narration, then automatically replace the voice with professional AI-generated audio.

**Key Features (MVP):**
- User authentication (Google OAuth + Email/Password)
- Payment-gated access (Pro $49/month, Scale $249/month)
- Chrome extension for screen recording (single tab + microphone)
- Video upload to AWS S3
- AI processing pipeline: OpenAI Whisper (transcription) → ElevenLabs (AI voice) → FFmpeg (audio replacement)
- Video playback and download
- Usage tracking and limit enforcement

**Timeline:** 12-16 days (1-2 developers)

**Deployment:** Production-ready for ~100-200 concurrent users

---

## Architecture

### System Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                          USER BROWSER                           │
│                                                                 │
│  ┌──────────────────┐              ┌───────────────────────┐  │
│  │   Next.js Web    │              │  Chrome Extension     │  │
│  │   Application    │◄────────────►│  (Screen Recorder)    │  │
│  └────────┬─────────┘              └──────────┬────────────┘  │
└───────────┼────────────────────────────────────┼───────────────┘
            │                                    │
            │ Auth, Upload, Download             │ Upload Video
            │                                    │
    ┌───────▼────────────────────────────────────▼──────────┐
    │              VERCEL (Frontend + APIs)                  │
    │                                                         │
    │  • Next.js App (SSR + Client)                          │
    │  • API Routes (Upload, Download, User Management)      │
    │  • Auth Middleware                                     │
    └───────┬─────────────────────────────────────┬──────────┘
            │                                     │
            │ Save to DB                          │ Queue job
            │                                     │
    ┌───────▼─────────┐              ┌───────────▼────────────┐
    │   SUPABASE      │              │   RAILWAY WORKER       │
    │                 │              │                        │
    │  • PostgreSQL   │              │  • Video Processing    │
    │  • Auth         │              │  • FFmpeg              │
    │  • RLS Policies │              │  • Whisper API         │
    └─────────────────┘              │  • ElevenLabs API      │
                                     └────────┬───────────────┘
                                              │
                                              │ Store videos
                                              │
                                     ┌────────▼───────────────┐
                                     │      AWS S3            │
                                     │                        │
                                     │  • Original videos     │
                                     │  • Processed videos    │
                                     │  • Thumbnails          │
                                     └────────────────────────┘
```

### Component Responsibilities

**Vercel (Main Application):**
- Frontend rendering (Next.js)
- User authentication (Supabase Auth)
- Video upload endpoint (quick, just saves to S3 and DB)
- Video download endpoint (generates S3 signed URLs)
- Payment webhook handling (Dodo Payments)
- User dashboard and UI

**Railway Worker (Processing Service):**
- Long-running video processing (2-3 minutes)
- FFmpeg operations (audio extraction, merging)
- External API calls (Whisper, ElevenLabs)
- Updates Supabase when complete
- Retry logic for failed jobs

**Supabase:**
- PostgreSQL database (users, videos, subscriptions)
- Authentication (Google OAuth, Email/Password)
- Row Level Security policies
- Real-time subscriptions (for status updates)

**AWS S3:**
- Video file storage (original and processed)
- Thumbnail images
- Pre-signed URLs for secure downloads

---

## Tech Stack

### Frontend
- **Framework:** Next.js 14 (App Router, TypeScript)
- **Styling:** Tailwind CSS
- **UI Components:** shadcn/ui (Radix UI primitives)
- **State Management:** React Context + Zustand
- **Video Player:** react-player

### Backend
- **Main App:** Vercel (Next.js API Routes)
- **Worker:** Railway (Node.js + Express)
- **Database:** Supabase (PostgreSQL)
- **Auth:** Supabase Auth
- **Storage:** AWS S3
- **Payments:** Dodo Payments

### Chrome Extension
- **Manifest:** V3
- **APIs:** chrome.tabCapture, chrome.storage
- **Communication:** chrome.runtime messaging + window.postMessage

### External APIs
- **Speech-to-Text:** OpenAI Whisper API ($0.006/minute)
- **Text-to-Speech:** ElevenLabs API (~$0.10 per video)

### Development Tools
- **Package Manager:** npm
- **Version Control:** Git
- **Deployment:** Vercel CLI, Railway CLI

---

## Implementation Phases

### Phase 1: Project Setup & Configuration
**Time:** 3-4 hours
**Dependencies:** None

#### 1.1 Initialize Next.js Project

**Tasks:**
1. Create Next.js project with TypeScript and Tailwind CSS
2. Install core dependencies
3. Set up project structure
4. Configure TypeScript and Tailwind

**Commands:**
```bash
npx create-next-app@latest zapra --typescript --tailwind --app --src-dir
cd zapra
```

**Install Dependencies:**
```bash
# Core
npm install @supabase/supabase-js @supabase/auth-helpers-nextjs @supabase/ssr

# AWS
npm install @aws-sdk/client-s3 @aws-sdk/s3-request-presigner

# State & Utils
npm install zustand axios date-fns

# UI Components
npm install clsx tailwind-merge lucide-react class-variance-authority
npm install @radix-ui/react-dialog @radix-ui/react-dropdown-menu @radix-ui/react-slot

# Video
npm install react-player

# OpenAI
npm install openai

# Dev Dependencies (for worker)
npm install -D fluent-ffmpeg @types/fluent-ffmpeg
```

**Folder Structure:**
```
zapra/
├── src/
│   ├── app/
│   │   ├── (auth)/
│   │   │   └── login/
│   │   ├── (dashboard)/
│   │   │   ├── layout.tsx
│   │   │   ├── page.tsx
│   │   │   ├── pricing/
│   │   │   ├── upgrade/
│   │   │   └── content/[id]/
│   │   ├── api/
│   │   │   ├── videos/
│   │   │   ├── payments/
│   │   │   └── auth/
│   │   ├── layout.tsx
│   │   └── page.tsx
│   ├── components/
│   │   ├── auth/
│   │   ├── dashboard/
│   │   ├── modals/
│   │   ├── video/
│   │   ├── pricing/
│   │   └── ui/
│   ├── lib/
│   │   ├── supabase/
│   │   ├── aws/
│   │   ├── payments/
│   │   └── utils.ts
│   ├── types/
│   │   ├── database.ts
│   │   └── plans.ts
│   ├── config/
│   │   └── pricing.ts
│   └── store/
├── chrome-extension/
│   ├── manifest.json
│   ├── background.js
│   ├── content.js
│   └── popup/
├── worker/ (Railway processing service)
│   ├── src/
│   │   ├── index.js
│   │   ├── processor.js
│   │   ├── whisper.js
│   │   └── elevenlabs.js
│   └── package.json
└── public/
```

#### 1.2 Environment Configuration

Create `.env.local`:
```env
# Supabase
NEXT_PUBLIC_SUPABASE_URL=your_supabase_project_url
NEXT_PUBLIC_SUPABASE_ANON_KEY=your_supabase_anon_key
SUPABASE_SERVICE_ROLE_KEY=your_service_role_key

# AWS S3
AWS_ACCESS_KEY_ID=your_aws_access_key
AWS_SECRET_ACCESS_KEY=your_aws_secret_key
AWS_REGION=us-east-1
AWS_S3_BUCKET=zapra-videos

# OpenAI
OPENAI_API_KEY=your_openai_api_key

# ElevenLabs
ELEVENLABS_API_KEY=your_elevenlabs_api_key
ELEVENLABS_VOICE_ID=your_default_voice_id

# Dodo Payments
DODO_API_KEY=your_dodo_api_key
DODO_WEBHOOK_SECRET=your_webhook_secret

# App
NEXT_PUBLIC_APP_URL=http://localhost:3000
WORKER_URL=http://localhost:4000
WORKER_API_KEY=your_secure_random_key_here

# Worker (.env for Railway)
SUPABASE_URL=same_as_above
SUPABASE_SERVICE_KEY=same_as_above
AWS_ACCESS_KEY_ID=same_as_above
AWS_SECRET_ACCESS_KEY=same_as_above
AWS_REGION=same_as_above
AWS_S3_BUCKET=same_as_above
OPENAI_API_KEY=same_as_above
ELEVENLABS_API_KEY=same_as_above
ELEVENLABS_VOICE_ID=same_as_above
WORKER_API_KEY=same_as_above
```

**Acceptance Criteria:**
- [ ] Next.js dev server runs on localhost:3000
- [ ] All dependencies installed without errors
- [ ] Environment variables configured
- [ ] TypeScript compiles with no errors

---

### Phase 2: Database Schema & Supabase Setup
**Time:** 2-3 hours
**Dependencies:** Phase 1 complete

#### 2.1 Database Schema

Create these tables in Supabase SQL Editor:

```sql
-- Enable UUID extension
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

-- Users table (handled by Supabase Auth, we just reference it)

-- Subscriptions table
CREATE TABLE subscriptions (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  user_id UUID NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
  plan TEXT NOT NULL CHECK (plan IN ('pro', 'scale')),
  status TEXT NOT NULL CHECK (status IN ('active', 'cancelled', 'past_due')),
  current_period_start TIMESTAMP WITH TIME ZONE NOT NULL,
  current_period_end TIMESTAMP WITH TIME ZONE NOT NULL,
  cancel_at_period_end BOOLEAN DEFAULT FALSE,
  dodo_subscription_id TEXT UNIQUE,
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  UNIQUE(user_id)
);

-- Usage tracking table
CREATE TABLE usage (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  user_id UUID NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
  period_start TIMESTAMP WITH TIME ZONE NOT NULL,
  period_end TIMESTAMP WITH TIME ZONE NOT NULL,
  videos_created INTEGER DEFAULT 0,
  videos_exported INTEGER DEFAULT 0,
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  UNIQUE(user_id, period_start)
);

-- Videos table
CREATE TABLE videos (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  user_id UUID NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
  title TEXT DEFAULT 'Untitled',
  status TEXT NOT NULL CHECK (status IN ('uploading', 'processing', 'completed', 'failed')),
  original_video_url TEXT,
  processed_video_url TEXT,
  thumbnail_url TEXT,
  transcript TEXT,
  duration INTEGER,
  error_message TEXT,
  retry_count INTEGER DEFAULT 0,
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Webhook events table (for idempotency)
CREATE TABLE webhook_events (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  event_id TEXT NOT NULL UNIQUE,
  event_type TEXT NOT NULL,
  payload JSONB,
  processed_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Upload tokens table (short-lived, single-use tokens for extension auth)
CREATE TABLE upload_tokens (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  token TEXT NOT NULL UNIQUE,
  user_id UUID NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
  expires_at TIMESTAMP WITH TIME ZONE NOT NULL,
  used BOOLEAN DEFAULT FALSE,
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Indexes for performance
CREATE INDEX idx_subscriptions_user_id ON subscriptions(user_id);
CREATE INDEX idx_subscriptions_status ON subscriptions(status);
CREATE INDEX idx_usage_user_period ON usage(user_id, period_start);
CREATE INDEX idx_videos_user_id ON videos(user_id);
CREATE INDEX idx_videos_status ON videos(status);
CREATE INDEX idx_videos_created_at ON videos(created_at DESC);
CREATE INDEX idx_webhook_events_event_id ON webhook_events(event_id);
CREATE INDEX idx_upload_tokens_token ON upload_tokens(token);
CREATE INDEX idx_upload_tokens_expires ON upload_tokens(expires_at);

-- Enable Row Level Security
ALTER TABLE subscriptions ENABLE ROW LEVEL SECURITY;
ALTER TABLE usage ENABLE ROW LEVEL SECURITY;
ALTER TABLE videos ENABLE ROW LEVEL SECURITY;
ALTER TABLE upload_tokens ENABLE ROW LEVEL SECURITY;

-- RLS Policies for subscriptions
CREATE POLICY "Users can view their own subscription"
  ON subscriptions FOR SELECT
  USING (auth.uid() = user_id);

CREATE POLICY "Users can insert their own subscription"
  ON subscriptions FOR INSERT
  WITH CHECK (auth.uid() = user_id);

CREATE POLICY "Users can update their own subscription"
  ON subscriptions FOR UPDATE
  USING (auth.uid() = user_id);

-- RLS Policies for usage
CREATE POLICY "Users can view their own usage"
  ON usage FOR SELECT
  USING (auth.uid() = user_id);

CREATE POLICY "System can manage usage"
  ON usage FOR ALL
  USING (true);

-- RLS Policies for videos
CREATE POLICY "Users can view their own videos"
  ON videos FOR SELECT
  USING (auth.uid() = user_id);

CREATE POLICY "Users can insert their own videos"
  ON videos FOR INSERT
  WITH CHECK (auth.uid() = user_id);

CREATE POLICY "Users can update their own videos"
  ON videos FOR UPDATE
  USING (auth.uid() = user_id);

CREATE POLICY "Users can delete their own videos"
  ON videos FOR DELETE
  USING (auth.uid() = user_id);

-- RLS Policies for upload_tokens
CREATE POLICY "Users can create their own upload tokens"
  ON upload_tokens FOR INSERT
  WITH CHECK (auth.uid() = user_id);

CREATE POLICY "Users can view their own upload tokens"
  ON upload_tokens FOR SELECT
  USING (auth.uid() = user_id);

-- Service role can manage all tokens (for cleanup and validation)
CREATE POLICY "Service role can manage upload tokens"
  ON upload_tokens FOR ALL
  USING (true);

-- Functions
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
  NEW.updated_at = NOW();
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- Triggers
CREATE TRIGGER update_subscriptions_updated_at
  BEFORE UPDATE ON subscriptions
  FOR EACH ROW
  EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER update_usage_updated_at
  BEFORE UPDATE ON usage
  FOR EACH ROW
  EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER update_videos_updated_at
  BEFORE UPDATE ON videos
  FOR EACH ROW
  EXECUTE FUNCTION update_updated_at_column();

-- Helper function to get current period usage
CREATE OR REPLACE FUNCTION get_current_period_usage(p_user_id UUID)
RETURNS TABLE(videos_created INTEGER, videos_exported INTEGER) AS $$
BEGIN
  RETURN QUERY
  SELECT
    COALESCE(u.videos_created, 0) as videos_created,
    COALESCE(u.videos_exported, 0) as videos_exported
  FROM usage u
  WHERE u.user_id = p_user_id
    AND u.period_start <= NOW()
    AND u.period_end > NOW()
  LIMIT 1;
END;
$$ LANGUAGE plpgsql;

-- CRITICAL: Atomic check-and-increment to prevent race conditions
-- This function prevents the race where two users at 49/50 both pass the check
-- Returns TRUE if video was created (under limit), FALSE if rejected (at limit)
CREATE OR REPLACE FUNCTION try_create_video(
  p_user_id UUID,
  p_max_videos INTEGER
)
RETURNS BOOLEAN AS $$
DECLARE
  period_start TIMESTAMP WITH TIME ZONE;
  period_end TIMESTAMP WITH TIME ZONE;
  current_count INTEGER;
  new_count INTEGER;
BEGIN
  -- Calculate current billing period
  SELECT s.current_period_start, s.current_period_end
  INTO period_start, period_end
  FROM subscriptions s
  WHERE s.user_id = p_user_id AND s.status = 'active';

  IF NOT FOUND THEN
    -- No active subscription
    RETURN FALSE;
  END IF;

  -- Atomic check + increment in single transaction
  -- IMPORTANT: This uses SELECT FOR UPDATE to lock the row
  INSERT INTO usage (user_id, period_start, period_end, videos_created, videos_exported)
  VALUES (p_user_id, period_start, period_end, 0, 0)
  ON CONFLICT (user_id, period_start)
  DO NOTHING;

  -- Lock the row and get current count
  SELECT videos_created INTO current_count
  FROM usage
  WHERE user_id = p_user_id AND period_start = period_start
  FOR UPDATE;

  -- Check limit
  IF current_count >= p_max_videos THEN
    -- At limit, reject
    RETURN FALSE;
  END IF;

  -- Under limit, increment
  UPDATE usage
  SET videos_created = videos_created + 1
  WHERE user_id = p_user_id AND period_start = period_start;

  RETURN TRUE;
END;
$$ LANGUAGE plpgsql;

-- Helper function to atomically increment export count
CREATE OR REPLACE FUNCTION increment_exports(p_user_id UUID)
RETURNS void AS $$
DECLARE
  period_start TIMESTAMP WITH TIME ZONE;
  period_end TIMESTAMP WITH TIME ZONE;
BEGIN
  -- Calculate current billing period
  SELECT s.current_period_start, s.current_period_end
  INTO period_start, period_end
  FROM subscriptions s
  WHERE s.user_id = p_user_id;

  -- Atomic increment using ON CONFLICT
  INSERT INTO usage (user_id, period_start, period_end, videos_created, videos_exported)
  VALUES (p_user_id, period_start, period_end, 0, 1)
  ON CONFLICT (user_id, period_start)
  DO UPDATE SET videos_exported = usage.videos_exported + 1;
END;
$$ LANGUAGE plpgsql;

-- Helper function to refund video quota on permanent failure
-- Called when a video fails permanently and user gets no usable output
CREATE OR REPLACE FUNCTION refund_video_quota(p_user_id UUID)
RETURNS void AS $$
DECLARE
  period_start TIMESTAMP WITH TIME ZONE;
  period_end TIMESTAMP WITH TIME ZONE;
BEGIN
  -- Calculate current billing period
  SELECT s.current_period_start, s.current_period_end
  INTO period_start, period_end
  FROM subscriptions s
  WHERE s.user_id = p_user_id;

  -- Decrement videos_created count (if exists and > 0)
  UPDATE usage
  SET videos_created = GREATEST(videos_created - 1, 0)
  WHERE user_id = p_user_id AND period_start = period_start;
END;
$$ LANGUAGE plpgsql;
```

#### 2.2 TypeScript Types

Create `src/types/database.ts`:

```typescript
export type PlanType = 'pro' | 'scale';
export type SubscriptionStatus = 'active' | 'cancelled' | 'past_due';
export type VideoStatus = 'uploading' | 'processing' | 'completed' | 'failed';

export interface Subscription {
  id: string;
  user_id: string;
  plan: PlanType;
  status: SubscriptionStatus;
  current_period_start: string;
  current_period_end: string;
  cancel_at_period_end: boolean;
  dodo_subscription_id: string | null;
  created_at: string;
  updated_at: string;
}

export interface Usage {
  id: string;
  user_id: string;
  period_start: string;
  period_end: string;
  videos_created: number;
  videos_exported: number;
  created_at: string;
  updated_at: string;
}

export interface Video {
  id: string;
  user_id: string;
  title: string;
  status: VideoStatus;
  original_video_url: string | null;
  processed_video_url: string | null;
  thumbnail_url: string | null;
  transcript: string | null;
  duration: number | null;
  error_message: string | null;
  retry_count: number;
  created_at: string;
  updated_at: string;
}

export interface Database {
  public: {
    Tables: {
      subscriptions: {
        Row: Subscription;
        Insert: Omit<Subscription, 'id' | 'created_at' | 'updated_at'>;
        Update: Partial<Omit<Subscription, 'id' | 'created_at' | 'updated_at'>>;
      };
      usage: {
        Row: Usage;
        Insert: Omit<Usage, 'id' | 'created_at' | 'updated_at'>;
        Update: Partial<Omit<Usage, 'id' | 'created_at' | 'updated_at'>>;
      };
      videos: {
        Row: Video;
        Insert: Omit<Video, 'id' | 'created_at' | 'updated_at'>;
        Update: Partial<Omit<Video, 'id' | 'created_at' | 'updated_at'>>;
      };
    };
  };
}
```

#### 2.3 Pricing Configuration

Create `src/config/pricing.ts`:

```typescript
export const PLANS = {
  pro: {
    id: 'pro',
    name: 'Pro',
    price: 49,
    currency: 'USD',
    interval: 'month',
    limits: {
      monthlyVideos: 50,
      maxVideoDuration: 15 * 60, // 15 minutes in seconds
      exports: 'unlimited' as const,
    },
    features: [
      '50 AI video recordings',
      'Unlimited video exports',
      'Recordings up to 15 minutes',
      'AI voice replacement',
      'HD quality exports',
    ],
  },
  scale: {
    id: 'scale',
    name: 'Scale',
    price: 249,
    currency: 'USD',
    interval: 'month',
    limits: {
      monthlyVideos: 300,
      maxVideoDuration: 30 * 60, // 30 minutes in seconds
      exports: 'unlimited' as const,
    },
    features: [
      '300 AI video recordings',
      'Unlimited video exports',
      'Recordings up to 30 minutes',
      'AI voice replacement',
      'HD quality exports',
      'Priority support',
    ],
  },
} as const;

export type PlanId = keyof typeof PLANS;

export function getPlanLimits(planId: PlanId) {
  return PLANS[planId].limits;
}

export function canUserRecord(
  plan: PlanId,
  currentUsage: number,
  videoDuration: number
): { allowed: boolean; reason?: string } {
  const limits = getPlanLimits(plan);

  if (currentUsage >= limits.monthlyVideos) {
    return {
      allowed: false,
      reason: `You've reached your monthly limit of ${limits.monthlyVideos} videos. Upgrade to continue.`,
    };
  }

  if (videoDuration > limits.maxVideoDuration) {
    return {
      allowed: false,
      reason: `Video duration (${Math.floor(videoDuration / 60)} min) exceeds your plan limit of ${limits.maxVideoDuration / 60} minutes.`,
    };
  }

  return { allowed: true };
}
```

#### 2.4 Supabase Client Setup

Create `src/lib/supabase/client.ts`:

```typescript
import { createBrowserClient } from '@supabase/ssr';
import type { Database } from '@/types/database';

export const createClient = () =>
  createBrowserClient<Database>(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!
  );
```

Create `src/lib/supabase/server.ts`:

**Note:** This uses Supabase SSR pattern for Next.js App Router. The cookie handling is non-obvious and required.

```typescript
import { createServerClient, type CookieOptions } from '@supabase/ssr';
import { cookies } from 'next/headers';
import type { Database } from '@/types/database';

export const createClient = () => {
  const cookieStore = cookies();

  return createServerClient<Database>(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
    {
      cookies: {
        get(name: string) {
          return cookieStore.get(name)?.value;
        },
        set(name: string, value: string, options: CookieOptions) {
          try {
            cookieStore.set({ name, value, ...options });
          } catch (error) {
            // Server component, can't set cookies
          }
        },
        remove(name: string, options: CookieOptions) {
          try {
            cookieStore.set({ name, value: '', ...options });
          } catch (error) {
            // Server component, can't remove cookies
          }
        },
      },
    }
  );
};
```

Create `src/lib/supabase/middleware.ts`:

```typescript
import { createServerClient, type CookieOptions } from '@supabase/ssr';
import { NextResponse, type NextRequest } from 'next/server';

export async function updateSession(request: NextRequest) {
  let response = NextResponse.next({
    request: {
      headers: request.headers,
    },
  });

  const supabase = createServerClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
    {
      cookies: {
        get(name: string) {
          return request.cookies.get(name)?.value;
        },
        set(name: string, value: string, options: CookieOptions) {
          request.cookies.set({ name, value, ...options });
          response = NextResponse.next({
            request: { headers: request.headers },
          });
          response.cookies.set({ name, value, ...options });
        },
        remove(name: string, options: CookieOptions) {
          request.cookies.set({ name, value: '', ...options });
          response = NextResponse.next({
            request: { headers: request.headers },
          });
          response.cookies.set({ name, value: '', ...options });
        },
      },
    }
  );

  await supabase.auth.getUser();
  return response;
}
```

Create `middleware.ts` in project root:

```typescript
import { type NextRequest } from 'next/server';
import { updateSession } from './src/lib/supabase/middleware';

export async function middleware(request: NextRequest) {
  return await updateSession(request);
}

export const config = {
  matcher: [
    '/((?!_next/static|_next/image|favicon.ico|.*\\.(?:svg|png|jpg|jpeg|gif|webp)$).*)',
  ],
};
```

**Acceptance Criteria:**
- [ ] Database tables created in Supabase
- [ ] RLS policies enabled and working
- [ ] TypeScript types defined
- [ ] Supabase clients connect successfully
- [ ] Pricing config is clear and modifiable

---

### Phase 3: Authentication
**Time:** 3-4 hours
**Dependencies:** Phase 2 complete

#### 3.1 Authentication Flow

**User Journey:**
1. User visits landing page
2. Clicks "Get Started"
3. Redirected to `/login`
4. Signs up with Google OAuth OR Email/Password
5. **No email verification required**
6. Redirected to dashboard
7. Sees empty dashboard with "Start Recording" prompt
8. Clicks "Start Recording" → Shown pricing plans
9. Must select plan and pay before recording

**Key Decision:** Payment validates identity (no email verification needed). Prevents bot abuse without friction for real users.

#### 3.2 Implementation Steps

**Files to Create:**

1. `src/app/(auth)/login/page.tsx` - Login page UI
2. `src/components/auth/LoginForm.tsx` - Login/signup form with tabs
3. `src/components/auth/GoogleButton.tsx` - Google OAuth button
4. `src/app/api/auth/callback/route.ts` - OAuth callback handler
5. `src/app/(dashboard)/layout.tsx` - Protected dashboard layout (checks auth)

**Key Patterns:**

**Server-side auth check:**
```typescript
// In dashboard layout
const supabase = createClient();
const { data: { user } } = await supabase.auth.getUser();

if (!user) {
  redirect('/login');
}
```

**Client-side auth actions:**
```typescript
// Google OAuth
const supabase = createClient();
await supabase.auth.signInWithOAuth({
  provider: 'google',
  options: {
    redirectTo: `${window.location.origin}/api/auth/callback`,
  },
});

// Email/Password signup
await supabase.auth.signUp({
  email,
  password,
  options: {
    emailRedirectTo: `${window.location.origin}/api/auth/callback`,
  },
});

// Email/Password login
await supabase.auth.signInWithPassword({ email, password });
```

**Supabase Configuration:**
- Enable Google OAuth in Supabase Dashboard: Authentication → Providers → Google
- Add OAuth credentials from Google Cloud Console
- Set redirect URL: `${APP_URL}/api/auth/callback`
- **Disable email confirmation** in Supabase: Authentication → Settings → "Confirm email" = OFF

**Password Reset:**
```typescript
// Request reset
await supabase.auth.resetPasswordForEmail(email, {
  redirectTo: `${window.location.origin}/reset-password`,
});

// Reset password page
await supabase.auth.updateUser({ password: newPassword });
```

**Design Reference:** `screenshots/02-login-page.png`
- Left side: Purple-pink gradient with Zapra logo
- Right side: White form with tabs (Sign Up / Login)
- Google button, email/password fields
- "Forgot Password?" link (only on Login tab)

**Testing:**
- [ ] Email signup creates user without verification
- [ ] Email login works immediately
- [ ] Google OAuth works and redirects correctly
- [ ] Session persists on page refresh
- [ ] Dashboard redirects to login if not authenticated
- [ ] Password reset email sends and works
- [ ] Sign out clears session

---

### Phase 4: Pricing & Payment Integration
**Time:** 4-5 hours
**Dependencies:** Phase 3 complete

#### 4.1 User Flow

**New User:**
```
Signup → Dashboard (empty state) → Click "Start Recording"
→ Redirect to /pricing → Select plan → Dodo Payments checkout
→ Webhook creates subscription → Redirect to dashboard → Can record
```

**Existing User (at limit):**
```
Try to record → Check usage → At limit
→ Show "Upgrade Required" modal → /upgrade page → Select higher plan
→ Dodo Payments → Webhook updates subscription → Can record
```

#### 4.2 Pricing Page

**Files to Create:**
1. `src/app/(public)/pricing/page.tsx` - Public pricing page
2. `src/components/pricing/PricingCard.tsx` - Reusable pricing card
3. `src/components/pricing/PricingComparison.tsx` - Feature comparison table

**Design Reference:** `screenshots/17-pricings.png`

**Layout:**
- Header: "Pricing" title with subtitle
- Two plans side-by-side: Pro ($49) and Scale ($249)
- Each card shows:
  - Plan name and price
  - "Get started" CTA button
  - Usage limits
  - Feature list with checkmarks
- Responsive: Stack on mobile

**Data Source:** Import from `src/config/pricing.ts`

#### 4.3 Upgrade Page (Logged-in Users)

**File:** `src/app/(dashboard)/upgrade/page.tsx`

**Shows:**
- Current plan and usage (e.g., "You've used 48/50 videos this month")
- Available plans (highlight recommended upgrade)
- "Upgrade Now" buttons

#### 4.4 Dodo Payments Integration

**Why Dodo Payments instead of Stripe:**
- **Geographic restriction:** Stripe is not available in India for new merchant accounts
- **MVP requirement:** Dodo Payments provides similar functionality with INR support
- **Migration path:** Can switch to Stripe later if expanding to markets where Stripe is preferred
- **Note:** Dodo Payments provides standard webhooks, checkout sessions, and subscription management similar to Stripe's API

**API Routes:**

1. `src/app/api/payments/create-checkout/route.ts`
   - Input: `{ plan: 'pro' | 'scale', userId: string }`
   - Output: Dodo checkout URL
   - Action: Create Dodo checkout session, return URL

2. `src/app/api/payments/webhook/route.ts`
   - Input: Dodo webhook payload
   - Verify webhook signature (DODO_WEBHOOK_SECRET)
   - Events to handle:
     - `checkout.completed` → Create subscription in DB
     - `subscription.renewed` → Update period dates
     - `subscription.cancelled` → Update status to 'cancelled'
     - `payment.failed` → Update status to 'past_due', send email

**Webhook Processing Logic:**

**CRITICAL:** Webhooks can fire multiple times (network retries, etc.). Must check for duplicate events to prevent double-charging.

```typescript
// Example webhook handler with idempotency
export async function POST(request: NextRequest) {
  const payload = await request.json();
  const signature = request.headers.get('dodo-signature');

  // Verify signature
  if (!verifyWebhookSignature(JSON.stringify(payload), signature)) {
    return NextResponse.json({ error: 'Invalid signature' }, { status: 401 });
  }

  const supabase = createClient();

  // IDEMPOTENCY CHECK: Have we already processed this event?
  const { data: existing } = await supabase
    .from('webhook_events')
    .select('id')
    .eq('event_id', payload.id)
    .single();

  if (existing) {
    console.log(`Webhook ${payload.id} already processed, skipping`);
    return NextResponse.json({ received: true }); // Already handled
  }

  // Record that we're processing this event
  await supabase.from('webhook_events').insert({
    event_id: payload.id,
    event_type: payload.type,
    payload: payload,
  });

  // Process the event
  if (payload.type === 'checkout.completed') {
    const { userId, plan, subscriptionId } = payload.data;

    // Create subscription
    await supabase.from('subscriptions').insert({
      user_id: userId,
      plan: plan,
      status: 'active',
      current_period_start: new Date(),
      current_period_end: addMonths(new Date(), 1),
      dodo_subscription_id: subscriptionId,
    });

    // Initialize usage tracking
    await supabase.from('usage').insert({
      user_id: userId,
      period_start: new Date(),
      period_end: addMonths(new Date(), 1),
      videos_created: 0,
      videos_exported: 0,
    });
  }

  return NextResponse.json({ received: true });
}
```

**Helper Functions:**

Create `src/lib/payments/dodo.ts`:
```typescript
export async function createCheckoutSession(plan: PlanId, userId: string) {
  // Call Dodo API to create checkout
  // Return checkout URL
}

export function verifyWebhookSignature(payload: string, signature: string): boolean {
  // Verify using DODO_WEBHOOK_SECRET
}
```

**Dodo Payments Setup:**
- Create Dodo account
- Get API keys
- Configure webhook URL: `${APP_URL}/api/payments/webhook`
- Create products in Dodo dashboard (Pro $49, Scale $249)
- Test with Dodo test mode

**Testing:**
- [ ] Pricing page displays correctly
- [ ] "Get started" button redirects to Dodo checkout
- [ ] Test payment completes successfully
- [ ] Webhook creates subscription in database
- [ ] User can access recording after payment
- [ ] Upgrade flow works (pro → scale)
- [ ] Subscription renewal updates period dates

---

### Phase 5: Dashboard & UI
**Time:** 4-5 hours
**Dependencies:** Phase 4 complete

#### 5.1 Dashboard Layout

**File:** `src/app/(dashboard)/layout.tsx`

**Components:**
- Sidebar (left, fixed width)
- Main content area (right, scrollable)

**Sidebar Items:**
- Logo
- Navigation: Home, Library (disabled), Shared Pages (disabled), Brand Kit (disabled), Knowledge Base (disabled)
- Upgrade section (shows plan, usage, "Upgrade Plan" button)
- User profile (avatar, email, logout)

**Design Reference:** `screenshots/03-dashboard-empty.png`, `screenshots/04-dashboard-with-videos.png`

#### 5.2 Home Page

**File:** `src/app/(dashboard)/page.tsx`

**Logic:**
```typescript
// Server component
const user = await getUser();
const subscription = await getSubscription(user.id);
const videos = await getVideos(user.id, { limit: 10 });

// Check if user has subscription
if (!subscription) {
  // Show: "Get started by choosing a plan"
  // CTA: "View Plans" → /pricing
}

// If has videos: Show video grid
// If no videos: Show welcome + quick guide
```

**Components:**

1. **Welcome Header**
   - First time: "Hi {name}, Welcome to Zapra 👋"
   - Returning: "Welcome back, {name}!"
   - "+ Create new" button (top right)

2. **Action Cards** (only show if has subscription)
   - "Start Recording" card (with icon)
   - "Upload a Video" card (disabled, greyed out)

3. **Quick Guide** (only show if no videos)
   - 4 steps with icons
   - Step 1: Download Chrome extension
   - Step 2: Record on any tab
   - Step 3: Generate AI content
   - Step 4: Export video

4. **Recent Content** (only show if has videos)
   - Video grid (3 columns)
   - "See all" link
   - Each video card shows: thumbnail, title, date, duration, language tag, status

**Files to Create:**
- `src/components/dashboard/Sidebar.tsx`
- `src/components/dashboard/StartRecordingCard.tsx`
- `src/components/dashboard/VideoCard.tsx`
- `src/components/dashboard/VideoGrid.tsx`

**Design Reference:** All dashboard screenshots

**Testing:**
- [ ] Sidebar renders correctly
- [ ] Navigation works (disabled items don't navigate)
- [ ] User can sign out
- [ ] Dashboard shows empty state for new users
- [ ] Dashboard shows video grid for users with videos
- [ ] Upgrade section shows current plan and usage
- [ ] Video cards display correctly

---

### Phase 6: Recording Flow & Modals
**Time:** 2-3 hours
**Dependencies:** Phase 5 complete

#### 6.1 Start Recording Flow

**User clicks "Start Recording" card:**

**Step 1: Check Subscription**
```typescript
const subscription = await getSubscription(user.id);

if (!subscription || subscription.status !== 'active') {
  // Redirect to /pricing
  router.push('/pricing');
  return;
}
```

**Step 2: Check Usage Limit**
```typescript
const usage = await getCurrentUsage(user.id);
const limits = getPlanLimits(subscription.plan);

if (usage.videos_created >= limits.monthlyVideos) {
  // Show "Upgrade Required" modal
  showUpgradeModal();
  return;
}
```

**Step 3: Check for Processing Videos**
```typescript
const processingVideos = await getVideos(user.id, { status: 'processing' });

if (processingVideos.length > 0) {
  // Show "Processing Blocked" modal
  showProcessingBlockedModal();
  return;
}
```

**Step 4: Show "Speak While Recording" Modal**
```typescript
// User sees prompt about speaking naturally
// "Got it, let's start!" button → Opens extension communication
```

#### 6.2 Modals

**Files to Create:**

1. `src/components/modals/SpeakWhileRecordingModal.tsx`
   - Design: `screenshots/05-start-recording-prompt.png`
   - Info icon, title, description
   - "Got it, let's start!" button

2. `src/components/modals/ProcessingBlockedModal.tsx`
   - Title: "Processing in Progress"
   - Message: "Please wait for your current video to finish processing before starting a new recording."
   - "OK" button

3. `src/components/modals/UpgradeRequiredModal.tsx`
   - Title: "Monthly Limit Reached"
   - Message: "You've used all {limit} videos this month. Upgrade to Scale for {scale_limit} videos."
   - "Upgrade Now" button → /upgrade
   - "Maybe Later" button

4. `src/components/modals/ExtensionNotInstalledModal.tsx`
   - Title: "Chrome Extension Required"
   - Message: "Install the Zapra extension to start recording."
   - "Install Extension" button → Chrome Web Store
   - Shows after user clicks "Got it, let's start!" if extension not detected

#### 6.3 Extension Communication

**IMPORTANT SECURITY NOTE:** Use short-lived, single-use upload tokens instead of storing long-lived auth tokens in the extension.

**Complete Token Exchange Flow:**

```
┌─────────────┐         ┌─────────────┐         ┌─────────────┐         ┌─────────────┐
│   Browser   │         │   Backend   │         │  Extension  │         │   AWS S3    │
│  (Web App)  │         │   (Vercel)  │         │ (Background)│         │             │
└─────┬───────┘         └──────┬──────┘         └──────┬──────┘         └──────┬──────┘
      │                        │                       │                       │
      │ 1. Click "Start Record"│                       │                       │
      │ (User authenticated)   │                       │                       │
      ├───────────────────────>│                       │                       │
      │ POST /api/videos/      │                       │                       │
      │ generate-upload-token  │                       │                       │
      │ (Supabase session      │                       │                       │
      │  cookie sent)          │                       │                       │
      │                        │                       │                       │
      │                        │ 2. Verify session    │                       │
      │                        │    Create upload_token│                       │
      │                        │    (5-min expiry,    │                       │
      │                        │     single-use)      │                       │
      │<───────────────────────│                       │                       │
      │ { uploadToken: "abc" } │                       │                       │
      │                        │                       │                       │
      │ 3. postMessage to ext  │                       │                       │
      │──────────────────────────────────────────────>│                       │
      │ { uploadToken, userId }│                       │                       │
      │                        │                       │                       │
      │                        │                       │ 4. User records video│
      │                        │                       │    (blob created)    │
      │                        │                       │                       │
      │                        │                       │ 5. Request presigned │
      │                        │<──────────────────────│    URL               │
      │                        │ POST /api/videos/     │                       │
      │                        │ request-upload        │                       │
      │                        │ { uploadToken }       │                       │
      │                        │                       │                       │
      │                        │ 6. Validate token    │                       │
      │                        │    Mark as used      │                       │
      │                        │    Create video DB   │                       │
      │                        │    Generate S3 URL   │                       │
      │                        │──────────────────────>│                       │
      │                        │ { videoId, presigned  │                       │
      │                        │   Url, s3Key }        │                       │
      │                        │                       │                       │
      │                        │                       │ 7. Upload directly   │
      │                        │                       │────────────────────>│
      │                        │                       │ PUT <presignedUrl>   │
      │                        │                       │ Body: video blob     │
      │                        │                       │                       │
      │                        │                       │<────────────────────│
      │                        │                       │ 200 OK               │
      │                        │                       │                       │
      │                        │                       │ 8. Notify completion│
      │                        │<──────────────────────│                       │
      │                        │ POST /api/videos/{id}/│                       │
      │                        │ complete-upload       │                       │
      │                        │ { s3Key, uploadToken }│                       │
      │                        │                       │                       │
      │                        │ 9. Increment usage   │                       │
      │                        │    Queue worker job  │                       │
      │                        │──────────────────────>│                       │
      │                        │ { success: true }     │                       │
      │                        │                       │                       │
      │<───────────────────────────────────────────────│ 10. Redirect user   │
      │ Redirect to /content/{id}/processing          │                       │
      │                        │                       │                       │
```

**Key Security Points:**
1. **Never pass Supabase JWT to extension** - Generate separate upload token
2. **Upload token is single-use** - Marked as `used=true` after first API call
3. **Short expiry** - 5 minutes is enough for recording + upload
4. **Origin verification** - Web app checks postMessage origin
5. **Token tied to user** - Backend validates token ownership

**In `StartRecordingCard.tsx`:**

```typescript
// When user clicks "Got it, let's start!"
async function handleStartRecording() {
  // Check if extension is installed
  window.postMessage({ type: 'ZAPRA_CHECK_EXTENSION' }, '*');

  // Listen for response
  window.addEventListener('message', (event) => {
    // SECURITY: Verify message origin
    const allowedOrigins = [
      'http://localhost:3000',
      'https://zapra.ai',
      'https://app.zapra.ai'
    ];

    if (!allowedOrigins.includes(event.origin)) {
      console.warn('Message from unauthorized origin:', event.origin);
      return;
    }

    if (event.data.type === 'ZAPRA_EXTENSION_READY') {
      // Extension installed, generate short-lived upload token
      generateUploadToken();
    } else if (event.data.type === 'ZAPRA_EXTENSION_NOT_FOUND') {
      // Show "Install Extension" modal with Chrome Store link
      showExtensionModal();
    }
  });
}

async function generateUploadToken() {
  // Call API to generate short-lived token (5 min expiry)
  const response = await fetch('/api/videos/generate-upload-token', {
    method: 'POST',
  });

  const { uploadToken } = await response.json();

  // Send to extension
  window.postMessage({
    type: 'ZAPRA_START_RECORDING',
    payload: { uploadToken, userId: user.id }
  }, '*');
}
```

**Create new API endpoint:** `src/app/api/videos/generate-upload-token/route.ts`

```typescript
import { NextResponse } from 'next/server';
import { createClient } from '@/lib/supabase/server';
import { randomBytes } from 'crypto';

export async function POST() {
  const supabase = createClient();

  const { data: { user } } = await supabase.auth.getUser();
  if (!user) {
    return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });
  }

  // Generate short-lived token (5 minute expiry)
  const uploadToken = randomBytes(32).toString('hex');
  const expiresAt = new Date(Date.now() + 5 * 60 * 1000); // 5 minutes

  // Store in database (create upload_tokens table)
  await supabase.from('upload_tokens').insert({
    token: uploadToken,
    user_id: user.id,
    expires_at: expiresAt,
    used: false,
  });

  return NextResponse.json({ uploadToken });
}
```

**Note:** The `upload_tokens` table and indexes are already created in Phase 2 database schema.

**Testing:**
- [ ] Subscription check works
- [ ] Usage limit check works
- [ ] Processing check works
- [ ] All modals display correctly
- [ ] Extension communication works (tested in Phase 7)

---

### Phase 7: Chrome Extension
**Time:** 5-6 hours
**Dependencies:** Phase 6 complete

#### 7.1 Extension Architecture

**Components:**
1. **Background Script** (background.js) - Handles recording, storage
2. **Content Script** (content.js) - Injected into pages, shows recording widget
3. **Popup** (popup.html/js) - Extension icon popup (optional for MVP)

**Communication Flow:**
```
Web App ←→ Content Script ←→ Background Script
   ↓              ↓                    ↓
window.postMessage | chrome.runtime | chrome.tabCapture
```

#### 7.2 Manifest

**File:** `chrome-extension/manifest.json`

```json
{
  "manifest_version": 3,
  "name": "Zapra Screen Recorder",
  "version": "1.0.0",
  "description": "Record your screen with AI-powered voice replacement",
  "permissions": [
    "activeTab",
    "tabCapture",
    "storage",
    "scripting"
  ],
  "host_permissions": [
    "http://localhost:3000/*",
    "https://*.zapra.ai/*"
  ],
  "background": {
    "service_worker": "background.js"
  },
  "content_scripts": [
    {
      "matches": ["http://localhost:3000/*", "https://*.zapra.ai/*"],
      "js": ["content.js"],
      "run_at": "document_end"
    }
  ],
  "action": {
    "default_title": "Zapra Screen Recorder"
  },
  "icons": {
    "16": "icons/icon16.png",
    "48": "icons/icon48.png",
    "128": "icons/icon128.png"
  }
}
```

#### 7.3 Content Script (Extension ↔ Web App Communication)

**File:** `chrome-extension/content.js`

**Responsibilities:**
- Listen for messages from web app
- Forward to background script
- Show/hide recording widget
- Handle recording countdown

**Key Message Types:**
- `ZAPRA_CHECK_EXTENSION` - Web app checking if extension installed
- `ZAPRA_START_RECORDING` - Web app requests recording start
- `ZAPRA_STOP_RECORDING` - User stops recording

**Pattern:**
```javascript
// Listen from web app
window.addEventListener('message', (event) => {
  // SECURITY: Verify origin
  const allowedOrigins = [
    'http://localhost:3000',
    'https://zapra.ai',
    'https://app.zapra.ai'
  ];

  if (!allowedOrigins.includes(event.origin)) {
    console.warn('Extension: Message from unauthorized origin:', event.origin);
    return;
  }

  if (event.data.type === 'ZAPRA_CHECK_EXTENSION') {
    // Respond that extension is installed
    window.postMessage({ type: 'ZAPRA_EXTENSION_READY' }, '*');
  }

  if (event.data.type === 'ZAPRA_START_RECORDING') {
    // Store upload token (short-lived, single-use)
    chrome.storage.local.set({
      uploadToken: event.data.payload.uploadToken,
      userId: event.data.payload.userId
    });

    // Forward to background script
    chrome.runtime.sendMessage({
      type: 'START_RECORDING',
      tabId: chrome.devtools?.inspectedWindow?.tabId
    });
  }

  if (event.data.type === 'ZAPRA_LOGOUT') {
    // Clear all stored user data on logout
    chrome.storage.local.clear(() => {
      console.log('Extension: Cleared user data on logout');
    });
  }
});

// Listen from background script
chrome.runtime.onMessage.addListener((message) => {
  if (message.type === 'SHOW_RECORDING_WIDGET') {
    showRecordingWidget();
  }

  if (message.type === 'RECORDING_COMPLETE') {
    hideRecordingWidget();
    // Clear upload token after use
    chrome.storage.local.remove(['uploadToken']);
    // Redirect to processing page
    window.location.href = `${APP_URL}/content/${message.videoId}/processing`;
  }

  if (message.type === 'RECORDING_ERROR') {
    hideRecordingWidget();
    // Clear upload token on error
    chrome.storage.local.remove(['uploadToken']);
    alert(message.error);
  }
});
```

**Recording Widget:**
- Small floating bar at top of page
- Shows: Timer, Pause button, Stop button, Delete button
- Design: `screenshots/11-recording-widget.png`

#### 7.3.1 Logout Integration (Web App → Extension)

**IMPORTANT:** When user logs out, the web app MUST notify the extension to clear stored credentials.

**Implementation in Dashboard Logout Handler:**

Add this to your logout button/function (e.g., in sidebar user profile component):

```typescript
// src/components/dashboard/UserProfile.tsx (or wherever logout is handled)
async function handleLogout() {
  // Notify extension to clear stored data (if extension is installed)
  window.postMessage({ type: 'ZAPRA_LOGOUT' }, '*');

  // Sign out from Supabase
  await supabase.auth.signOut();

  // Redirect to login
  router.push('/login');
}
```

**What this does:**
- Extension content script receives `ZAPRA_LOGOUT` message
- Clears all chrome.storage.local data (uploadToken, userId)
- Prevents unauthorized access if same browser/extension is used by different user

**Security note:** This prevents the scenario where:
1. User A logs in and uses extension (stores credentials)
2. User A logs out
3. User B logs in on same browser
4. User B tries to record → would incorrectly use User A's stale credentials

#### 7.4 Background Script (Recording Logic)

**File:** `chrome-extension/background.js`

**Responsibilities:**
- Capture screen + microphone
- Record to WebM blob
- Upload to S3 via API
- Handle errors and retries

**Recording Flow:**

**Step 1: Get Media Stream**
```javascript
// Request tab selection (Chrome's built-in UI)
chrome.tabs.query({ active: true }, (tabs) => {
  chrome.tabCapture.capture({
    audio: true,
    video: true
  }, (stream) => {
    // Got screen stream
    startRecording(stream);
  });
});
```

**Step 2: Add Microphone**
```javascript
async function addMicrophone(screenStream) {
  const micStream = await navigator.mediaDevices.getUserMedia({ audio: true });

  // Mix audio tracks
  const audioContext = new AudioContext();
  const screenAudio = audioContext.createMediaStreamSource(
    new MediaStream(screenStream.getAudioTracks())
  );
  const micAudio = audioContext.createMediaStreamSource(micStream);
  const dest = audioContext.createMediaStreamDestination();

  screenAudio.connect(dest);
  micAudio.connect(dest);

  // Combine video + mixed audio
  return new MediaStream([
    ...screenStream.getVideoTracks(),
    ...dest.stream.getAudioTracks()
  ]);
}
```

**Step 3: Record**
```javascript
let mediaRecorder;
let chunks = [];

function startRecording(stream) {
  mediaRecorder = new MediaRecorder(stream, {
    mimeType: 'video/webm;codecs=vp9,opus'
  });

  mediaRecorder.ondataavailable = (event) => {
    if (event.data.size > 0) {
      chunks.push(event.data);
    }
  };

  mediaRecorder.onstop = async () => {
    const blob = new Blob(chunks, { type: 'video/webm' });
    await uploadVideo(blob);
  };

  mediaRecorder.start();

  // Notify content script to show widget
  chrome.tabs.sendMessage(tabId, { type: 'SHOW_RECORDING_WIDGET' });
}

function stopRecording() {
  if (mediaRecorder && mediaRecorder.state !== 'inactive') {
    mediaRecorder.stop();
    // Stop all tracks
    mediaRecorder.stream.getTracks().forEach(track => track.stop());
  }
}
```

**Step 4: Upload**
```javascript
async function uploadVideo(blob) {
  try {
    // Get auth token from storage
    const { authToken, userId } = await chrome.storage.local.get(['authToken', 'userId']);

    // Create form data
    const formData = new FormData();
    formData.append('video', blob, `recording-${Date.now()}.webm`);

    // Upload to API
    const response = await fetch(`${APP_URL}/api/videos/upload`, {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${authToken}`
      },
      body: formData
    });

    const data = await response.json();

    if (data.success) {
      // Notify content script
      chrome.tabs.sendMessage(tabId, {
        type: 'RECORDING_COMPLETE',
        videoId: data.videoId
      });
    } else {
      throw new Error(data.error);
    }
  } catch (error) {
    console.error('Upload failed:', error);
    // Show error notification
    chrome.notifications.create({
      type: 'basic',
      title: 'Upload Failed',
      message: 'Failed to upload video. Please try again.',
    });
  }
}
```

**Extension Post-Install Flow:**

When user installs extension from Chrome Web Store:
1. Chrome Web Store allows setting "redirect URL"
2. Set redirect to: `${APP_URL}/extension-installed`
3. Create page `src/app/extension-installed/page.tsx`
4. Page detects extension and sends auth token automatically
5. Shows success message: "Extension ready! You can start recording."

**Testing:**
- [ ] Extension installs in Chrome
- [ ] Content script detects extension
- [ ] Auth token successfully passed from app to extension
- [ ] Tab selection UI appears
- [ ] Recording starts successfully
- [ ] Recording widget appears and functions
- [ ] Stop button ends recording
- [ ] Video uploads successfully
- [ ] User redirected to processing page

---

### Phase 8: Video Upload & Download API
**Time:** 2-3 hours
**Dependencies:** Phase 7 complete

#### 8.1 Upload Flow (Presigned URL Strategy)

**CRITICAL:** Extension uploads directly to S3 using presigned URLs to avoid Vercel body size limits (4.5MB default, 50MB max on paid plans). Videos can be hundreds of MB.

**New Upload Flow:**
1. Extension requests presigned S3 URL from API (with upload token)
2. API validates token, creates video record, generates presigned URL
3. Extension uploads directly to S3 using presigned URL
4. Extension calls completion API with video ID
5. API atomically increments usage and queues worker job

#### 8.2 Step 1: Get Presigned URL Endpoint

**File:** `src/app/api/videos/request-upload/route.ts`

```typescript
import { NextRequest, NextResponse } from 'next/server';
import { createClient } from '@/lib/supabase/server';
import { generatePresignedUploadUrl, generateVideoKey } from '@/lib/aws/s3';
import { getPlanLimits } from '@/config/pricing';

export async function POST(request: NextRequest) {
  try {
    const { uploadToken, fileSize, duration } = await request.json();

    if (!uploadToken) {
      return NextResponse.json({ error: 'Upload token required' }, { status: 400 });
    }

    const supabase = createClient();

    // Verify upload token (short-lived, single-use) with atomic update to prevent race condition
    const { data: tokenData, error: tokenError } = await supabase
      .from('upload_tokens')
      .update({ used: true })
      .eq('token', uploadToken)
      .eq('used', false) // Only update if not already used (atomic check-and-set)
      .select('user_id, expires_at')
      .single();

    // If no rows were updated, token was either already used, doesn't exist, or is invalid
    if (tokenError || !tokenData) {
      return NextResponse.json({ error: 'Invalid, expired, or already used token' }, { status: 401 });
    }

    // Check expiration
    if (new Date(tokenData.expires_at) < new Date()) {
      return NextResponse.json({ error: 'Token has expired' }, { status: 401 });
    }

    const userId = tokenData.user_id;

    // Check subscription
    const { data: subscription } = await supabase
      .from('subscriptions')
      .select('*')
      .eq('user_id', userId)
      .single();

    if (!subscription || subscription.status !== 'active') {
      return NextResponse.json({ error: 'No active subscription' }, { status: 403 });
    }

    // Check usage limits
    const { data: usage } = await supabase.rpc('get_current_period_usage', {
      p_user_id: userId
    });

    const limits = getPlanLimits(subscription.plan);

    if (usage && usage.videos_created >= limits.monthlyVideos) {
      return NextResponse.json({ error: 'Monthly limit reached' }, { status: 403 });
    }

    // FILE SIZE LIMIT: Prevent storage blow-up
    const MAX_FILE_SIZE = 500 * 1024 * 1024; // 500 MB for MVP
    if (fileSize && fileSize > MAX_FILE_SIZE) {
      return NextResponse.json({
        error: 'File too large. Maximum size is 500 MB.',
        maxSize: MAX_FILE_SIZE
      }, { status: 413 });
    }

    // DURATION LIMIT: Check against plan limits
    if (duration && duration > limits.maxVideoDuration) {
      return NextResponse.json({
        error: `Video too long. Your plan allows ${limits.maxVideoDuration / 60} minutes.`,
        maxDuration: limits.maxVideoDuration
      }, { status: 413 });
    }

    // Create video record (status: 'uploading')
    const { data: video } = await supabase
      .from('videos')
      .insert({
        user_id: userId,
        title: 'Untitled',
        status: 'uploading',
      })
      .select()
      .single();

    // Generate S3 key and presigned URL
    const s3Key = generateVideoKey(userId, video.id, 'original');
    const presignedUrl = await generatePresignedUploadUrl(s3Key, 'video/webm');

    return NextResponse.json({
      success: true,
      videoId: video.id,
      presignedUrl,
      s3Key,
    });

  } catch (error) {
    console.error('Upload request error:', error);
    return NextResponse.json({ error: 'Internal server error' }, { status: 500 });
  }
}
```

#### 8.3 Step 2: Complete Upload Endpoint

**File:** `src/app/api/videos/[id]/complete-upload/route.ts`

```typescript
import { NextRequest, NextResponse } from 'next/server';
import { createClient } from '@/lib/supabase/server';
import { getPlanLimits } from '@/config/pricing';

/**
 * Retry helper with exponential backoff for worker API calls
 * Prevents silent failures when worker temporarily unavailable
 */
async function retryWithBackoff<T>(
  fn: () => Promise<T>,
  maxRetries: number = 3,
  delayMs: number = 2000
): Promise<T> {
  let lastError: Error | undefined;

  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
      return await fn();
    } catch (error) {
      lastError = error as Error;
      console.error(`[Retry ${attempt}/${maxRetries}] Worker call failed:`, error);

      if (attempt < maxRetries) {
        const waitTime = delayMs * Math.pow(2, attempt - 1); // Exponential backoff: 2s, 4s, 8s
        console.log(`[Retry ${attempt}/${maxRetries}] Waiting ${waitTime}ms before retry...`);
        await new Promise(resolve => setTimeout(resolve, waitTime));
      }
    }
  }

  throw lastError || new Error('Max retries reached');
}

export async function POST(
  request: NextRequest,
  { params }: { params: { id: string } }
) {
  try {
    const { s3Key, uploadToken } = await request.json();
    const videoId = params.id;

    const supabase = createClient();

    // Verify upload token was used for this video
    const { data: tokenData } = await supabase
      .from('upload_tokens')
      .select('user_id')
      .eq('token', uploadToken)
      .eq('used', true)
      .single();

    if (!tokenData) {
      return NextResponse.json({ error: 'Invalid token' }, { status: 401 });
    }

    // Get video and verify ownership
    const { data: video } = await supabase
      .from('videos')
      .select('*')
      .eq('id', videoId)
      .eq('user_id', tokenData.user_id)
      .single();

    if (!video) {
      return NextResponse.json({ error: 'Video not found' }, { status: 404 });
    }

    // Get user's plan limits
    const { data: subscription } = await supabase
      .from('subscriptions')
      .select('plan')
      .eq('user_id', tokenData.user_id)
      .single();

    if (!subscription) {
      return NextResponse.json({ error: 'No active subscription' }, { status: 403 });
    }

    const limits = getPlanLimits(subscription.plan);

    // CRITICAL: Atomic check + increment to prevent race conditions
    // This prevents two users at 49/50 from both creating video #50
    const { data: allowed, error: rpcError } = await supabase.rpc('try_create_video', {
      p_user_id: tokenData.user_id,
      p_max_videos: limits.monthlyVideos
    });

    if (rpcError) {
      console.error('try_create_video error:', rpcError);
      return NextResponse.json({ error: 'Failed to check usage limit' }, { status: 500 });
    }

    if (!allowed) {
      // User hit their limit between upload and completion
      // Delete the video record and return error
      await supabase.from('videos').delete().eq('id', videoId);
      return NextResponse.json({
        error: 'Monthly video limit reached',
        limit: limits.monthlyVideos
      }, { status: 403 });
    }

    // Construct S3 URL
    const s3Url = `https://${process.env.AWS_S3_BUCKET}.s3.${process.env.AWS_REGION}.amazonaws.com/${s3Key}`;

    // Update video with S3 URL and status
    await supabase
      .from('videos')
      .update({
        original_video_url: s3Url,
        status: 'processing',
      })
      .eq('id', videoId);

    // Queue job on worker (with authentication + retry logic)
    // Retry 3 times with exponential backoff (2s, 4s, 8s delays)
    await retryWithBackoff(async () => {
      const response = await fetch(`${process.env.WORKER_URL}/process`, {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'x-worker-api-key': process.env.WORKER_API_KEY!
        },
        body: JSON.stringify({ videoId }),
      });

      if (!response.ok) {
        throw new Error(`Worker responded with status ${response.status}`);
      }

      return response;
    });

    return NextResponse.json({ success: true, videoId });

  } catch (error) {
    console.error('Complete upload error:', error);
    return NextResponse.json({ error: 'Internal server error' }, { status: 500 });
  }
}
```

#### 8.4 S3 Utilities

Create `src/lib/aws/s3.ts`:

```typescript
import { S3Client, PutObjectCommand, GetObjectCommand } from '@aws-sdk/client-s3';
import { getSignedUrl } from '@aws-sdk/s3-request-presigner';

const s3 = new S3Client({
  region: process.env.AWS_REGION!,
  credentials: {
    accessKeyId: process.env.AWS_ACCESS_KEY_ID!,
    secretAccessKey: process.env.AWS_SECRET_ACCESS_KEY!,
  },
});

const BUCKET = process.env.AWS_S3_BUCKET!;
const REGION = process.env.AWS_REGION!;

// Generate presigned URL for uploading (extension → S3 direct)
export async function generatePresignedUploadUrl(
  key: string,
  contentType: string
): Promise<string> {
  const command = new PutObjectCommand({
    Bucket: BUCKET,
    Key: key,
    ContentType: contentType,
  });

  // URL expires in 10 minutes (enough time for large uploads)
  return getSignedUrl(s3, command, { expiresIn: 600 });
}

// Generate presigned URL for downloading
export async function getDownloadUrl(key: string): Promise<string> {
  const command = new GetObjectCommand({
    Bucket: BUCKET,
    Key: key,
  });

  // URL expires in 1 hour
  return getSignedUrl(s3, command, { expiresIn: 3600 });
}

// Upload from worker (for processed videos)
export async function uploadToS3(
  buffer: Buffer,
  key: string,
  contentType: string
): Promise<string> {
  await s3.send(new PutObjectCommand({
    Bucket: BUCKET,
    Key: key,
    Body: buffer,
    ContentType: contentType,
  }));

  return `https://${BUCKET}.s3.${REGION}.amazonaws.com/${key}`;
}

export function generateVideoKey(
  userId: string,
  videoId: string,
  type: 'original' | 'processed'
): string {
  // Original videos are WebM (from Chrome MediaRecorder)
  // Processed videos are MP4 (H.264 for Safari compatibility)
  const extension = type === 'original' ? 'webm' : 'mp4';
  return `videos/${userId}/${videoId}/${type}-${Date.now()}.${extension}`;
}
```

#### 8.5 Update Extension Background Script

Update `chrome-extension/background.js` upload function:

```javascript
async function uploadVideo(blob) {
  try {
    // Get upload token from storage
    const { uploadToken, userId } = await chrome.storage.local.get(['uploadToken', 'userId']);

    if (!uploadToken) {
      throw new Error('No upload token found');
    }

    // Step 1: Request presigned URL
    // Include file size and duration for validation
    const fileSize = blob.size;
    const duration = recordingDuration; // Track this during recording

    const requestResponse = await fetch(`${APP_URL}/api/videos/request-upload`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        uploadToken,
        fileSize,
        duration // in seconds
      }),
    });

    if (!requestResponse.ok) {
      throw new Error('Failed to get upload URL');
    }

    const { videoId, presignedUrl, s3Key } = await requestResponse.json();

    // Step 2: Upload directly to S3 using presigned URL
    const uploadResponse = await fetch(presignedUrl, {
      method: 'PUT',
      headers: {
        'Content-Type': 'video/webm',
      },
      body: blob,
    });

    if (!uploadResponse.ok) {
      throw new Error('Failed to upload to S3');
    }

    // Step 3: Notify API that upload is complete
    const completeResponse = await fetch(`${APP_URL}/api/videos/${videoId}/complete-upload`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ s3Key, uploadToken }),
    });

    if (!completeResponse.ok) {
      throw new Error('Failed to complete upload');
    }

    // Success! Notify content script
    chrome.tabs.sendMessage(tabId, {
      type: 'RECORDING_COMPLETE',
      videoId: videoId
    });

  } catch (error) {
    console.error('Upload failed:', error);
    chrome.tabs.sendMessage(tabId, {
      type: 'RECORDING_ERROR',
      error: error.message
    });
  }
}
```

#### 8.2 Download Endpoint

**File:** `src/app/api/videos/[id]/download/route.ts`

**Flow:**
1. Verify user owns video
2. Generate S3 signed URL (expires in 1 hour)
3. Return URL (browser auto-downloads)

```typescript
export async function GET(
  request: NextRequest,
  { params }: { params: { id: string } }
) {
  const supabase = createClient();

  // Check auth
  const { data: { user } } = await supabase.auth.getUser();
  if (!user) {
    return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });
  }

  // Get video
  const { data: video } = await supabase
    .from('videos')
    .select('*')
    .eq('id', params.id)
    .eq('user_id', user.id)
    .single();

  if (!video || !video.processed_video_url) {
    return NextResponse.json({ error: 'Video not found' }, { status: 404 });
  }

  // Extract S3 key from URL
  const url = new URL(video.processed_video_url);
  const key = url.pathname.slice(1); // Remove leading /

  // Generate signed URL
  const downloadUrl = await getDownloadUrl(key);

  // Increment export count
  await supabase.rpc('increment_exports', { p_user_id: user.id });

  // Redirect to signed URL (triggers download)
  return NextResponse.redirect(downloadUrl);
}
```

**Testing:**
- [ ] Upload endpoint accepts video files
- [ ] Auth check works (rejects invalid tokens)
- [ ] Subscription check works
- [ ] Usage limit enforcement works
- [ ] Video uploads to S3 successfully
- [ ] DB record created correctly
- [ ] Download endpoint generates valid URLs
- [ ] Download actually downloads the file

---

### Phase 9: Railway Worker (Video Processing)
**Time:** 6-8 hours
**Dependencies:** Phase 8 complete

#### 9.1 Worker Architecture

**Why Railway:**
- No serverless timeouts (can process for minutes)
- FFmpeg pre-installed
- Simple deployment
- Persistent server

**Worker Responsibilities:**
1. Listen for processing jobs
2. Download video from S3
3. Extract audio using FFmpeg
4. Transcribe audio with Whisper
5. Generate AI voice with ElevenLabs
6. Replace audio in video using FFmpeg
7. Generate thumbnail (first frame)
8. Upload processed video and thumbnail to S3
9. Update database
10. Retry on failure (up to 3 attempts)

**IMPORTANT - Video Format Strategy:**
- **Original videos:** WebM (VP9 + Opus) - Chrome MediaRecorder default format
- **Processed videos:** MP4 (H.264 + AAC) - Cross-browser compatible, works on Safari/iOS
- **Why change format:** Safari and iOS don't support WebM playback
- **FFmpeg handles conversion:** Transcode during audio replacement step

#### 9.2 Worker Setup

**Folder:** `worker/`

**File:** `worker/package.json`

**IMPORTANT:** Worker uses TypeScript for type safety and consistency with frontend.

```json
{
  "name": "zapra-worker",
  "version": "1.0.0",
  "type": "module",
  "scripts": {
    "dev": "ts-node-esm src/index.ts",
    "build": "tsc",
    "start": "node dist/index.js"
  },
  "dependencies": {
    "express": "^4.18.2",
    "@supabase/supabase-js": "^2.39.0",
    "@aws-sdk/client-s3": "^3.490.0",
    "@aws-sdk/s3-request-presigner": "^3.490.0",
    "openai": "^4.24.1",
    "axios": "^1.6.0",
    "fluent-ffmpeg": "^2.1.2"
  },
  "devDependencies": {
    "typescript": "^5.2.2",
    "ts-node": "^10.9.1",
    "@types/node": "^20.5.1",
    "@types/express": "^4.17.17",
    "@types/fluent-ffmpeg": "^2.1.21"
  }
}
```

**File:** `worker/tsconfig.json`
```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ES2022",
    "lib": ["ES2022"],
    "moduleResolution": "node",
    "esModuleInterop": true,
    "strict": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "outDir": "./dist",
    "rootDir": "./src"
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules"]
}
```

**File:** `worker/src/index.ts` (Express server)

**Note:** All worker files use `.ts` extension for TypeScript.

```typescript
import express from 'express';
import { processVideo, retryFailedJobs, checkStuckVideos } from './processor';

const app = express();
app.use(express.json());

const PORT = process.env.PORT || 4000;
const WORKER_API_KEY = process.env.WORKER_API_KEY;

// Authentication middleware
const authenticateWorker = (req, res, next) => {
  const apiKey = req.headers['x-worker-api-key'];

  if (!apiKey || apiKey !== WORKER_API_KEY) {
    return res.status(401).json({ error: 'Unauthorized - Invalid API key' });
  }

  next();
};

// Health check (no auth required)
app.get('/health', (req, res) => {
  res.json({ status: 'ok' });
});

// Process video endpoint (requires authentication)
app.post('/process', authenticateWorker, async (req, res) => {
  const { videoId } = req.body;

  if (!videoId) {
    return res.status(400).json({ error: 'videoId required' });
  }

  // Start processing in background
  processVideo(videoId).catch(console.error);

  // Return immediately
  res.json({ success: true, message: 'Processing started' });
});

// Start server
app.listen(PORT, () => {
  console.log(`Worker listening on port ${PORT}`);
});

// RETRY SYSTEM: Run every 10 minutes to retry failed jobs
setInterval(() => {
  retryFailedJobs().catch(console.error);
}, 10 * 60 * 1000); // 10 minutes

// Run once on startup
retryFailedJobs().catch(console.error);

// STUCK VIDEO POLLER: Run every 30 seconds to catch silent worker failures
setInterval(() => {
  checkStuckVideos().catch(console.error);
}, 30 * 1000); // 30 seconds

// Run once on startup
checkStuckVideos().catch(console.error);
```

#### 9.2.5 Worker Configuration

**File:** `worker/src/config.ts`

**IMPORTANT:** Duplicate plan configuration to avoid broken import path when worker is deployed separately.

```typescript
/**
 * Plan limits configuration for worker
 *
 * CRITICAL: This is duplicated from src/config/pricing.ts to avoid import issues
 * when worker runs in separate deployment. Keep these values in sync!
 *
 * If you update plan limits in the main app, update them here too.
 */
export const PLANS = {
  pro: {
    monthlyVideos: 50,
    maxVideoDuration: 15 * 60, // 15 minutes in seconds
  },
  scale: {
    monthlyVideos: 300,
    maxVideoDuration: 30 * 60, // 30 minutes in seconds
  },
} as const;

export type PlanId = keyof typeof PLANS;

export function getPlanLimits(planId: PlanId) {
  return PLANS[planId];
}
```

#### 9.3 Video Processor

**File:** `worker/src/processor.ts`

**Main Processing Function:**

```typescript
import { createClient } from '@supabase/supabase-js';
import { downloadFromS3, uploadToS3 } from './s3';
import { extractAudio, replaceAudio, extractThumbnail, getVideoDuration, syncAudioToVideo, getAudioDuration } from './ffmpeg';
import { transcribeAudio } from './whisper';
import { generateVoice } from './elevenlabs';
import { getPlanLimits } from './config'; // Use local worker config
import fs from 'fs';
import path from 'path';
import os from 'os';

const supabase = createClient(
  process.env.SUPABASE_URL!,
  process.env.SUPABASE_SERVICE_KEY!
);

export async function processVideo(videoId: string): Promise<void> {
  const tempDir = path.join(os.tmpdir(), `zapra-${videoId}`);
  fs.mkdirSync(tempDir, { recursive: true });

  try {
    console.log(`[${videoId}] Starting processing`);

    // Get video from DB
    const { data: video } = await supabase
      .from('videos')
      .select('*')
      .eq('id', videoId)
      .single();

    if (!video) {
      throw new Error('Video not found');
    }

    // IDEMPOTENCY CHECK: Skip if already completed
    if (video.status === 'completed') {
      console.log(`[${videoId}] Already completed, skipping`);
      return;
    }

    // IDEMPOTENCY CHECK: Only process if status is 'processing'
    if (video.status !== 'processing') {
      console.log(`[${videoId}] Status is ${video.status}, skipping`);
      return;
    }

    // PRE-FLIGHT USAGE CHECK: Verify user still has credits before processing
    // (Subscription could have expired between upload and processing)
    const { data: subscription } = await supabase
      .from('subscriptions')
      .select('status, plan, current_period_end')
      .eq('user_id', video.user_id)
      .single();

    if (!subscription || subscription.status !== 'active') {
      console.error(`[${videoId}] User subscription inactive during processing`);
      await supabase
        .from('videos')
        .update({
          status: 'failed',
          error_message: 'Subscription expired before processing could complete',
        })
        .eq('id', videoId);
      return;
    }

    // Check if subscription period is still valid
    if (new Date(subscription.current_period_end) < new Date()) {
      console.error(`[${videoId}] User subscription period ended`);
      await supabase
        .from('videos')
        .update({
          status: 'failed',
          error_message: 'Subscription period ended before processing could complete',
        })
        .eq('id', videoId);
      return;
    }

    // Download original video
    const videoPath = path.join(tempDir, 'original.webm');
    // Extract S3 key from URL: https://bucket.s3.region.amazonaws.com/key -> key
    const s3Key = new URL(video.original_video_url).pathname.slice(1); // Remove leading /
    await downloadFromS3(s3Key, videoPath);

    // Get duration
    const duration = await getVideoDuration(videoPath);

    // CRITICAL: Verify video duration against plan limits
    // (Extension sends duration but could lie - verify server-side)
    const { data: limits } = await supabase
      .from('subscriptions')
      .select('plan')
      .eq('user_id', video.user_id)
      .single();

    if (limits) {
      const planLimits = getPlanLimits(limits.plan);
      if (duration > planLimits.maxVideoDuration) {
        console.error(`[${videoId}] Video duration (${duration}s) exceeds plan limit (${planLimits.maxVideoDuration}s)`);
        await supabase
          .from('videos')
          .update({
            status: 'failed',
            error_message: `Video duration (${Math.floor(duration / 60)} min) exceeds your plan limit of ${planLimits.maxVideoDuration / 60} minutes. Please upgrade or trim your video.`,
          })
          .eq('id', videoId);
        return;
      }
    }

    // Extract audio
    console.log(`[${videoId}] Extracting audio`);
    const audioPath = path.join(tempDir, 'audio.mp3');
    await extractAudio(videoPath, audioPath);

    // Transcribe with Whisper
    console.log(`[${videoId}] Transcribing audio`);
    const transcript = await transcribeAudio(audioPath);

    if (!transcript || transcript.trim().length === 0) {
      throw new Error('No audio detected in video');
    }

    // Save transcript to DB
    await supabase
      .from('videos')
      .update({ transcript })
      .eq('id', videoId);

    // Generate AI voice with ElevenLabs
    console.log(`[${videoId}] Generating AI voice`);
    const aiVoicePath = path.join(tempDir, 'ai-voice.mp3');
    await generateVoice(transcript, aiVoicePath);

    // CRITICAL: Synchronize AI audio duration with video duration
    console.log(`[${videoId}] Synchronizing audio duration with video`);
    const aiAudioDuration = await getAudioDuration(aiVoicePath);
    const syncedAudioPath = path.join(tempDir, 'ai-voice-synced.mp3');

    if (Math.abs(duration - aiAudioDuration) > 0.5) {
      // Duration differs by more than 0.5 seconds - needs sync
      console.log(`[${videoId}] AI audio (${aiAudioDuration}s) != video (${duration}s), syncing...`);
      await syncAudioToVideo(aiVoicePath, syncedAudioPath, duration);
    } else {
      // Duration close enough, use as-is
      console.log(`[${videoId}] AI audio duration matches, no sync needed`);
      fs.copyFileSync(aiVoicePath, syncedAudioPath);
    }

    // Replace audio
    console.log(`[${videoId}] Replacing audio in video`);
    const processedPath = path.join(tempDir, 'processed.mp4');
    await replaceAudio(videoPath, syncedAudioPath, processedPath);

    // Generate thumbnail
    console.log(`[${videoId}] Generating thumbnail`);
    const thumbnailPath = path.join(tempDir, 'thumbnail.jpg');
    await extractThumbnail(videoPath, thumbnailPath);

    // Upload processed video
    const processedBuffer = fs.readFileSync(processedPath);
    const processedKey = `videos/${video.user_id}/${videoId}/processed-${Date.now()}.mp4`;
    const processedUrl = await uploadToS3(processedBuffer, processedKey, 'video/mp4');

    // Upload thumbnail
    const thumbnailBuffer = fs.readFileSync(thumbnailPath);
    const thumbnailKey = `videos/${video.user_id}/${videoId}/thumbnail.jpg`;
    const thumbnailUrl = await uploadToS3(thumbnailBuffer, thumbnailKey, 'image/jpeg');

    // Update video status
    await supabase
      .from('videos')
      .update({
        status: 'completed',
        processed_video_url: processedUrl,
        thumbnail_url: thumbnailUrl,
        duration: Math.floor(duration),
      })
      .eq('id', videoId);

    console.log(`[${videoId}] Processing complete`);

  } catch (error) {
    console.error(`[${videoId}] Processing failed:`, error);

    // Retry logic
    const { data: video } = await supabase
      .from('videos')
      .select('retry_count')
      .eq('id', videoId)
      .single();

    if (video.retry_count < 3) {
      // Retry immediately with exponential backoff
      console.log(`[${videoId}] Retrying... (attempt ${video.retry_count + 1})`);

      await supabase
        .from('videos')
        .update({ retry_count: video.retry_count + 1 })
        .eq('id', videoId);

      // Exponential backoff: 5s, 15s, 45s
      const delay = Math.pow(3, video.retry_count) * 5000;
      setTimeout(() => processVideo(videoId), delay);

    } else {
      // Max retries reached - mark as failed AND refund quota
      console.error(`[${videoId}] Max retries reached, marking as failed and refunding quota`);

      // Get video to retrieve user_id
      const { data: failedVideo } = await supabase
        .from('videos')
        .select('user_id')
        .eq('id', videoId)
        .single();

      // Mark video as failed
      await supabase
        .from('videos')
        .update({
          status: 'failed',
          error_message: error.message,
          updated_at: new Date().toISOString(),
        })
        .eq('id', videoId);

      // Refund quota since user got no usable output
      if (failedVideo?.user_id) {
        console.log(`[${videoId}] Refunding quota for user ${failedVideo.user_id}`);
        await supabase.rpc('refund_video_quota', { p_user_id: failedVideo.user_id });
      }
    }

  } finally {
    // Cleanup temp files
    fs.rmSync(tempDir, { recursive: true, force: true });
  }
}

/**
 * Retry system for failed jobs
 * Run this every 10 minutes to retry failed videos
 */
export async function retryFailedJobs() {
  console.log('[Retry System] Checking for failed jobs...');

  // Find failed videos that haven't exceeded retry limit
  const { data: failedVideos } = await supabase
    .from('videos')
    .select('*')
    .eq('status', 'failed')
    .lt('retry_count', 3)
    .order('updated_at', { ascending: true })
    .limit(10); // Process 10 at a time

  if (!failedVideos || failedVideos.length === 0) {
    console.log('[Retry System] No failed jobs to retry');
    return;
  }

  console.log(`[Retry System] Found ${failedVideos.length} failed jobs to retry`);

  for (const video of failedVideos) {
    // Reset status to processing
    await supabase
      .from('videos')
      .update({
        status: 'processing',
        retry_count: video.retry_count + 1,
      })
      .eq('id', video.id);

    // Queue for processing
    processVideo(video.id).catch(console.error);

    // Small delay between retries to avoid overwhelming system
    await new Promise(resolve => setTimeout(resolve, 1000));
  }
}

/**
 * Check for stuck videos (videos in 'processing' status for >5 minutes)
 * Run this every 30 seconds to catch silent worker failures
 */
export async function checkStuckVideos() {
  console.log('[Stuck Video Check] Checking for stuck videos...');

  // Find videos stuck in 'processing' for more than 5 minutes
  const fiveMinutesAgo = new Date(Date.now() - 5 * 60 * 1000).toISOString();

  const { data: stuckVideos } = await supabase
    .from('videos')
    .select('*')
    .eq('status', 'processing')
    .lt('updated_at', fiveMinutesAgo)
    .lt('retry_count', 3)
    .order('updated_at', { ascending: true })
    .limit(10); // Process 10 at a time

  if (!stuckVideos || stuckVideos.length === 0) {
    console.log('[Stuck Video Check] No stuck videos found');
    return;
  }

  console.log(`[Stuck Video Check] Found ${stuckVideos.length} stuck videos, reprocessing...`);

  for (const video of stuckVideos) {
    console.log(`[Stuck Video Check] Reprocessing video ${video.id} (stuck since ${video.updated_at})`);

    // Update retry count and status
    await supabase
      .from('videos')
      .update({
        status: 'processing',
        retry_count: video.retry_count + 1,
        updated_at: new Date().toISOString(), // Update timestamp to prevent re-polling
      })
      .eq('id', video.id);

    // Queue for processing
    processVideo(video.id).catch(console.error);

    // Small delay between retries to avoid overwhelming system
    await new Promise(resolve => setTimeout(resolve, 1000));
  }
}
```

#### 9.4 FFmpeg Utilities

**File:** `worker/src/ffmpeg.ts`

**SECURITY NOTE:** fluent-ffmpeg handles command escaping internally, but we still validate all file paths to prevent injection attacks.

```typescript
import ffmpeg from 'fluent-ffmpeg';
import path from 'path';
import fs from 'fs';
import os from 'os';

/**
 * Sanitize and validate file paths to prevent shell injection
 */
function validateFilePath(filePath: string): string {
  // Check for shell control characters
  const dangerousChars = /[;&|`$<>(){}[\]!]/;
  if (dangerousChars.test(filePath)) {
    throw new Error(`Invalid file path: contains dangerous characters`);
  }

  // Ensure path is absolute and normalized
  const normalized = path.normalize(path.resolve(filePath));

  // Prevent path traversal (must stay within OS temp dir)
  // IMPORTANT: This must match where processor.ts creates temp directories (os.tmpdir())
  const tempDir = path.resolve(os.tmpdir());
  if (!normalized.startsWith(tempDir)) {
    throw new Error(`Invalid file path: outside allowed directory (${tempDir})`);
  }

  return normalized;
}

/**
 * Validate that file exists before passing to FFmpeg
 */
function ensureFileExists(filePath: string): void {
  if (!fs.existsSync(filePath)) {
    throw new Error(`File does not exist: ${filePath}`);
  }
}

export function extractAudio(videoPath: string, audioPath: string): Promise<void> {
  // SECURITY: Validate all file paths
  validateFilePath(videoPath);
  validateFilePath(audioPath);
  ensureFileExists(videoPath);
  return new Promise((resolve, reject) => {
    ffmpeg(videoPath)
      .output(audioPath)
      .audioCodec('libmp3lame')
      .noVideo()
      .on('end', () => resolve())
      .on('error', (err) => reject(err))
      .run();
  });
}

export function replaceAudio(videoPath: string, audioPath: string, outputPath: string): Promise<void> {
  // SECURITY: Validate all file paths
  validateFilePath(videoPath);
  validateFilePath(audioPath);
  validateFilePath(outputPath);
  ensureFileExists(videoPath);
  ensureFileExists(audioPath);

  return new Promise((resolve, reject) => {
    ffmpeg()
      .input(videoPath)
      .input(audioPath)
      .outputOptions([
        '-c:v libx264',     // H.264 video codec (Safari compatible)
        '-preset fast',     // Fast encoding preset
        '-crf 23',          // Quality (lower = better, 18-28 is good range)
        '-c:a aac',         // AAC audio codec
        '-b:a 128k',        // Audio bitrate
        '-map 0:v:0',       // Use video from first input
        '-map 1:a:0',       // Use audio from second input
        '-shortest',        // End when shortest stream ends
        '-movflags +faststart', // Enable fast start for web streaming
      ])
      .output(outputPath)
      .on('end', () => resolve())
      .on('error', (err) => reject(err))
      .run();
  });
}

export function extractThumbnail(videoPath: string, thumbnailPath: string): Promise<void> {
  // SECURITY: Validate all file paths
  validateFilePath(videoPath);
  validateFilePath(thumbnailPath);
  ensureFileExists(videoPath);

  return new Promise((resolve, reject) => {
    ffmpeg(videoPath)
      .screenshots({
        timestamps: ['1'], // At 1 second
        filename: path.basename(thumbnailPath),
        folder: path.dirname(thumbnailPath),
        size: '640x360'
      })
      .on('end', () => resolve())
      .on('error', (err) => reject(err));
  });
}

export function getVideoDuration(videoPath: string): Promise<number> {
  // SECURITY: Validate file path
  validateFilePath(videoPath);
  ensureFileExists(videoPath);

  return new Promise((resolve, reject) => {
    ffmpeg.ffprobe(videoPath, (err, metadata) => {
      if (err) reject(err);
      else resolve(metadata.format.duration || 0);
    });
  });
}

/**
 * Get audio file duration
 */
export function getAudioDuration(audioPath: string): Promise<number> {
  // SECURITY: Validate file path
  validateFilePath(audioPath);
  ensureFileExists(audioPath);

  return new Promise((resolve, reject) => {
    ffmpeg.ffprobe(audioPath, (err, metadata) => {
      if (err) reject(err);
      else resolve(metadata.format.duration || 0);
    });
  });
}

/**
 * CRITICAL: Synchronize AI audio duration with video duration
 * Handles the common case where AI-generated voice is shorter/longer than original
 *
 * Strategy:
 * - If AI audio is shorter: Pad with silence at the end
 * - If AI audio is 1-20% longer: Speed up slightly using atempo filter
 * - If AI audio is >20% longer: Fail with error (transcript mismatch)
 */
export function syncAudioToVideo(inputAudioPath: string, outputAudioPath: string, targetDuration: number): Promise<void> {
  // SECURITY: Validate file paths
  validateFilePath(inputAudioPath);
  validateFilePath(outputAudioPath);
  ensureFileExists(inputAudioPath);

  return new Promise(async (resolve, reject) => {
    try {
      const currentDuration = await getAudioDuration(inputAudioPath);
      const difference = targetDuration - currentDuration;
      const percentDiff = Math.abs(difference / targetDuration) * 100;

      console.log(`Audio sync: current=${currentDuration}s, target=${targetDuration}s, diff=${difference}s (${percentDiff.toFixed(1)}%)`);

      if (percentDiff > 20) {
        // Duration mismatch too large - likely wrong transcript
        reject(new Error(
          `AI audio duration mismatch too large (${percentDiff.toFixed(1)}%). ` +
          `This usually means the transcript doesn't match the video content.`
        ));
        return;
      }

      if (difference > 0.5) {
        // AI audio is shorter - pad with silence
        console.log(`Padding audio with ${difference}s of silence`);

        ffmpeg(inputAudioPath)
          .audioFilters([
            `apad=pad_dur=${difference}` // Add silence at end
          ])
          .duration(targetDuration) // Ensure exact duration
          .output(outputAudioPath)
          .on('end', () => resolve())
          .on('error', (err) => reject(err))
          .run();

      } else if (difference < -0.5) {
        // AI audio is longer - speed up slightly
        const speedFactor = currentDuration / targetDuration;
        console.log(`Speeding up audio by factor of ${speedFactor.toFixed(3)}`);

        // atempo filter has limits: 0.5 to 2.0
        if (speedFactor < 0.5 || speedFactor > 2.0) {
          reject(new Error(`Speed adjustment factor ${speedFactor} out of range`));
          return;
        }

        ffmpeg(inputAudioPath)
          .audioFilters([
            `atempo=${speedFactor}` // Speed up/slow down
          ])
          .output(outputAudioPath)
          .on('end', () => resolve())
          .on('error', (err) => reject(err))
          .run();

      } else {
        // Duration close enough - just copy
        fs.copyFileSync(inputAudioPath, outputAudioPath);
        resolve();
      }

    } catch (err) {
      reject(err);
    }
  });
}
```

#### 9.5 Whisper Integration

**File:** `worker/src/whisper.ts`

```typescript
import OpenAI from 'openai';
import fs from 'fs';

const openai = new OpenAI({
  apiKey: process.env.OPENAI_API_KEY,
});

/**
 * Exponential backoff retry utility
 */
async function retryWithBackoff<T>(
  fn: () => Promise<T>,
  maxRetries: number = 3,
  baseDelay: number = 1000
): Promise<T> {
  for (let attempt = 0; attempt < maxRetries; attempt++) {
    try {
      return await fn();
    } catch (error) {
      const isLastAttempt = attempt === maxRetries - 1;

      // Don't retry on certain errors
      if (error.status === 400 || error.status === 401 || error.status === 403) {
        throw error; // Authentication/bad request errors won't be fixed by retrying
      }

      if (isLastAttempt) {
        throw error;
      }

      // Exponential backoff: 1s, 2s, 4s, 8s...
      const delay = baseDelay * Math.pow(2, attempt);
      const jitter = Math.random() * 1000; // Add jitter to prevent thundering herd

      console.log(`Retry attempt ${attempt + 1}/${maxRetries} after ${delay + jitter}ms`);
      await new Promise(resolve => setTimeout(resolve, delay + jitter));
    }
  }
}

export async function transcribeAudio(audioPath: string): Promise<string> {
  return retryWithBackoff(async () => {
    const audioFile = fs.createReadStream(audioPath);

    const transcription = await openai.audio.transcriptions.create({
      file: audioFile,
      model: 'whisper-1',
      language: 'en',
    });

    return transcription.text;
  }, 3, 1000); // 3 retries, 1 second base delay
}
```

#### 9.6 ElevenLabs Integration

**File:** `worker/src/elevenlabs.ts`

```typescript
import axios, { AxiosError } from 'axios';
import fs from 'fs';

const ELEVENLABS_API_KEY = process.env.ELEVENLABS_API_KEY!;
const VOICE_ID = process.env.ELEVENLABS_VOICE_ID!;

/**
 * Exponential backoff retry utility (same as whisper.ts)
 */
async function retryWithBackoff<T>(
  fn: () => Promise<T>,
  maxRetries: number = 3,
  baseDelay: number = 1000
): Promise<T> {
  for (let attempt = 0; attempt < maxRetries; attempt++) {
    try {
      return await fn();
    } catch (error) {
      const isLastAttempt = attempt === maxRetries - 1;

      // Don't retry on authentication/bad request errors
      const axiosError = error as AxiosError;
      if (axiosError.response?.status === 400 ||
          axiosError.response?.status === 401 ||
          axiosError.response?.status === 403) {
        throw error;
      }

      if (isLastAttempt) {
        throw error;
      }

      // Exponential backoff with jitter
      const delay = baseDelay * Math.pow(2, attempt);
      const jitter = Math.random() * 1000;

      console.log(`ElevenLabs retry ${attempt + 1}/${maxRetries} after ${delay + jitter}ms`);
      await new Promise(resolve => setTimeout(resolve, delay + jitter));
    }
  }
  throw new Error('Max retries exceeded');
}

export async function generateVoice(text: string, outputPath: string): Promise<void> {
  return retryWithBackoff(async () => {
    const response = await axios.post(
      `https://api.elevenlabs.io/v1/text-to-speech/${VOICE_ID}`,
      {
        text,
        model_id: 'eleven_monolingual_v1',
        voice_settings: {
          stability: 0.5,
          similarity_boost: 0.75,
        },
      },
      {
        headers: {
          'xi-api-key': ELEVENLABS_API_KEY,
          'Content-Type': 'application/json',
        },
        responseType: 'arraybuffer',
        timeout: 30000, // 30 second timeout
      }
    );

    fs.writeFileSync(outputPath, response.data);
  }, 3, 1000); // 3 retries, 1 second base delay
}
```

#### 9.7 S3 Utilities (Worker)

**File:** `worker/src/s3.ts`

```typescript
import { S3Client, PutObjectCommand, GetObjectCommand } from '@aws-sdk/client-s3';
import fs from 'fs';

const s3 = new S3Client({
  region: process.env.AWS_REGION!,
  credentials: {
    accessKeyId: process.env.AWS_ACCESS_KEY_ID!,
    secretAccessKey: process.env.AWS_SECRET_ACCESS_KEY!,
  },
});

const BUCKET = process.env.AWS_S3_BUCKET!;

/**
 * Download file from S3 using credentials (not public URL)
 * IMPORTANT: Cannot use fetch() because bucket policy forbids public GET
 */
export async function downloadFromS3(s3Key: string, outputPath: string): Promise<void> {
  const command = new GetObjectCommand({
    Bucket: BUCKET,
    Key: s3Key,
  });

  const response = await s3.send(command);

  // Convert readable stream to buffer
  const chunks: Uint8Array[] = [];
  for await (const chunk of response.Body as any) {
    chunks.push(chunk);
  }
  const buffer = Buffer.concat(chunks);

  fs.writeFileSync(outputPath, buffer);
}

export async function uploadToS3(buffer: Buffer, key: string, contentType: string): Promise<string> {
  await s3.send(new PutObjectCommand({
    Bucket: BUCKET,
    Key: key,
    Body: buffer,
    ContentType: contentType,
  }));

  return `https://${BUCKET}.s3.${process.env.AWS_REGION}.amazonaws.com/${key}`;
}
```

#### 9.8 Railway Deployment

**IMPORTANT: FFmpeg Setup**

Railway uses Nixpacks by default, which includes FFmpeg. However, to be explicit and ensure the correct version:

**Option 1: Use Nixpacks (Recommended)**

Create `railway.toml`:
```toml
[build]
builder = "nixpacks"

[deploy]
startCommand = "npm start"
restartPolicyType = "on-failure"
```

Railway's Nixpacks automatically detects Node.js projects and includes FFmpeg. Verify after deployment:
```bash
railway run ffmpeg -version
```

**Option 2: Use Custom Dockerfile (If FFmpeg issues occur)**

Create `worker/Dockerfile`:
```dockerfile
FROM node:18-slim

# Install FFmpeg and dependencies
RUN apt-get update && apt-get install -y \
    ffmpeg \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app

# Copy package files
COPY package*.json ./

# Install dependencies
RUN npm ci --only=production

# Copy source code
COPY . .

# Verify FFmpeg is installed
RUN ffmpeg -version

EXPOSE 4000

CMD ["npm", "start"]
```

Update `railway.toml`:
```toml
[build]
builder = "dockerfile"
dockerfilePath = "Dockerfile"

[deploy]
startCommand = "npm start"
restartPolicyType = "on-failure"
```

**Deploy Steps:**
1. Install Railway CLI: `npm install -g @railway/cli`
2. Login: `railway login`
3. Create project: `railway init`
4. Link to folder: `cd worker && railway link`
5. Set env vars: `railway variables set SUPABASE_URL=... OPENAI_API_KEY=...` (etc.)
6. Deploy: `railway up`
7. Get URL: `railway domain` (e.g., `zapra-worker.up.railway.app`)
8. Update main app's `.env.local`: `WORKER_URL=https://zapra-worker.up.railway.app`

**Testing:**
- [ ] Worker deploys successfully on Railway
- [ ] Health check endpoint responds
- [ ] Receives processing job from main app
- [ ] Downloads video from S3
- [ ] Extracts audio with FFmpeg
- [ ] Transcribes with Whisper
- [ ] Generates AI voice with ElevenLabs
- [ ] Replaces audio successfully
- [ ] Generates thumbnail
- [ ] Uploads processed video and thumbnail to S3
- [ ] Updates database status to 'completed'
- [ ] Retry logic works on failure
- [ ] Temp files cleaned up

---

### Phase 10: Video Pages
**Time:** 3-4 hours
**Dependencies:** Phase 9 complete

#### 10.1 Processing Page

**File:** `src/app/(dashboard)/content/[id]/processing/page.tsx`

**Purpose:** Show animated loading state while video processes (~2-3 minutes)

**Logic:**
- Subscribe to real-time updates from Supabase for instant status changes (no polling!)
- If status = 'completed', redirect to result page
- If status = 'failed', show error message
- If no audio detected error, show specific message: "No audio detected. Please re-record with voice narration."

**Benefits of Real-time vs Polling:**
- ✅ Instant updates (0ms vs 3s delay)
- ✅ 99% less database load (1 query vs continuous polling)
- ✅ Better UX (user sees completion immediately)
- ✅ Free (included in Supabase)

**Implementation:**

```typescript
'use client';

import { useEffect, useState } from 'react';
import { useRouter } from 'next/navigation';
import { createClient } from '@/lib/supabase/client';
import { ProcessingAnimation } from '@/components/video/ProcessingAnimation';
import { RealtimeChannel } from '@supabase/supabase-js';

export default function ProcessingPage({ params }: { params: { id: string } }) {
  const router = useRouter();
  const supabase = createClient();
  const [error, setError] = useState<string | null>(null);
  const [retryCount, setRetryCount] = useState(0);

  useEffect(() => {
    let channel: RealtimeChannel;

    // Initial status check
    const checkInitialStatus = async () => {
      const { data: video } = await supabase
        .from('videos')
        .select('status, error_message, retry_count')
        .eq('id', params.id)
        .single();

      if (video?.status === 'completed') {
        router.push(`/content/${params.id}`);
      } else if (video?.status === 'failed') {
        setError(video.error_message || 'Processing failed');
        setRetryCount(video.retry_count || 0);
      }
    };

    checkInitialStatus();

    // Subscribe to real-time updates for this specific video
    channel = supabase
      .channel(`video-${params.id}`)
      .on(
        'postgres_changes',
        {
          event: 'UPDATE',
          schema: 'public',
          table: 'videos',
          filter: `id=eq.${params.id}`
        },
        (payload) => {
          const updatedVideo = payload.new as { status: string; error_message?: string; retry_count?: number };

          if (updatedVideo.status === 'completed') {
            // Video processing complete - redirect to result page
            router.push(`/content/${params.id}`);
          } else if (updatedVideo.status === 'failed') {
            // Video processing failed - show error
            setError(updatedVideo.error_message || 'Processing failed');
            setRetryCount(updatedVideo.retry_count || 0);
          }
        }
      )
      .subscribe();

    // Cleanup subscription on unmount
    return () => {
      channel.unsubscribe();
    };
  }, [params.id, router, supabase]);

  if (error) {
    return <ErrorDisplay error={error} retryCount={retryCount} videoId={params.id} />;
  }

  return (
    <div className="flex items-center justify-center min-h-screen bg-white">
      <ProcessingAnimation />
    </div>
  );
}
```

**Component:** `src/components/video/ProcessingAnimation.tsx`

**Design Reference:** `screenshots/13-processing-animation.png`
- Animated planet illustration (use Lottie or CSS animation)
- Text: "Processing your recording"
- Subtext: "This may take a few moments..."
- Loading dots animation

**Lottie Animation:**
- Find free Lottie animation: lottiefiles.com
- Search "loading planet" or "processing"
- Download JSON, save to `public/lottie/processing.json`
- Use `lottie-react` package: `npm install lottie-react`

**Component:** `src/components/video/ErrorDisplay.tsx`

**Purpose:** Show user-friendly error messages with retry options

```typescript
'use client';

import { useState } from 'react';
import { createClient } from '@/lib/supabase/client';
import { Button } from '@/components/ui/button';
import { AlertCircle, RefreshCw } from 'lucide-react';

interface ErrorDisplayProps {
  error: string;
  retryCount: number;
  videoId: string;
}

export function ErrorDisplay({ error, retryCount, videoId }: ErrorDisplayProps) {
  const [retrying, setRetrying] = useState(false);
  const supabase = createClient();

  // Determine user-friendly error message
  const getUserFriendlyMessage = () => {
    if (error.includes('No audio detected')) {
      return {
        title: 'No Audio Detected',
        description: 'Your recording didn\'t contain any audio. Please record again and speak clearly during the recording.',
        canRetry: false,
      };
    }

    if (error.includes('timeout') || error.includes('timed out')) {
      return {
        title: 'Processing Timeout',
        description: 'Your video took too long to process. This usually happens with very long videos or poor internet connection.',
        canRetry: retryCount < 3,
      };
    }

    if (error.includes('API') || error.includes('Whisper') || error.includes('ElevenLabs')) {
      return {
        title: 'API Error',
        description: 'There was a temporary issue with our AI service. We\'ll automatically retry this video.',
        canRetry: retryCount < 3,
      };
    }

    // Generic error
    return {
      title: 'Processing Failed',
      description: 'Something went wrong while processing your video. Please try recording again.',
      canRetry: retryCount < 3,
    };
  };

  const handleRetry = async () => {
    setRetrying(true);

    try {
      // Reset video status to trigger retry
      await supabase
        .from('videos')
        .update({
          status: 'processing',
          error_message: null,
        })
        .eq('id', videoId);

      // Trigger worker retry
      await fetch(`${process.env.NEXT_PUBLIC_WORKER_URL}/process`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ videoId }),
      });

      // Redirect back to processing page
      window.location.href = `/content/${videoId}/processing`;
    } catch (err) {
      console.error('Retry failed:', err);
      alert('Failed to retry. Please contact support.');
      setRetrying(false);
    }
  };

  const message = getUserFriendlyMessage();

  return (
    <div className="flex items-center justify-center min-h-screen bg-gray-50">
      <div className="max-w-md w-full bg-white rounded-lg shadow-lg p-8">
        <div className="flex items-center justify-center w-16 h-16 bg-red-100 rounded-full mx-auto mb-4">
          <AlertCircle className="w-8 h-8 text-red-600" />
        </div>

        <h2 className="text-2xl font-bold text-center text-gray-900 mb-2">
          {message.title}
        </h2>

        <p className="text-gray-600 text-center mb-6">
          {message.description}
        </p>

        {retryCount > 0 && (
          <p className="text-sm text-gray-500 text-center mb-4">
            Retry attempts: {retryCount}/3
          </p>
        )}

        <div className="flex gap-3">
          {message.canRetry && (
            <Button
              onClick={handleRetry}
              disabled={retrying}
              className="flex-1 bg-purple-600 hover:bg-purple-700"
            >
              {retrying ? (
                <>
                  <RefreshCw className="w-4 h-4 mr-2 animate-spin" />
                  Retrying...
                </>
              ) : (
                <>
                  <RefreshCw className="w-4 h-4 mr-2" />
                  Retry Processing
                </>
              )}
            </Button>
          )}

          <Button
            variant="outline"
            onClick={() => window.location.href = '/'}
            className="flex-1"
          >
            Back to Dashboard
          </Button>
        </div>

        {!message.canRetry && (
          <Button
            className="w-full mt-3 bg-purple-600 hover:bg-purple-700"
            onClick={() => window.location.href = '/'}
          >
            Record New Video
          </Button>
        )}

        {/* Technical error details (collapsed by default) */}
        <details className="mt-6 text-xs text-gray-500">
          <summary className="cursor-pointer">Technical Details</summary>
          <pre className="mt-2 p-2 bg-gray-100 rounded overflow-x-auto">
            {error}
          </pre>
        </details>
      </div>
    </div>
  );
}
```

#### 10.2 Video Result Page

**File:** `src/app/(dashboard)/content/[id]/page.tsx`

**Purpose:** Display processed video with playback and download options

**Design Reference:** `screenshots/15-result-page.png`

**Layout:**
- Header bar with: Back button, Video/Document tabs, title, action buttons
- Action buttons: Edit (disabled), Translate (disabled), Delete (disabled), Export Video
- Video player in center (large, with gradient background)

**Implementation:**

```typescript
import { createClient } from '@/lib/supabase/server';
import { redirect } from 'next/navigation';
import { VideoPlayer } from '@/components/video/VideoPlayer';
import { Download, Edit, Trash2 } from 'lucide-react';
import { Button } from '@/components/ui/button';

export default async function VideoResultPage({ params }: { params: { id: string } }) {
  const supabase = createClient();

  const { data: { user } } = await supabase.auth.getUser();
  if (!user) redirect('/login');

  const { data: video } = await supabase
    .from('videos')
    .select('*')
    .eq('id', params.id)
    .eq('user_id', user.id)
    .single();

  if (!video) redirect('/');
  if (video.status === 'processing') redirect(`/content/${params.id}/processing`);

  return (
    <div className="min-h-screen bg-gray-50">
      {/* Header */}
      <div className="bg-white border-b">
        <div className="max-w-7xl mx-auto px-8 py-4">
          <div className="flex items-center justify-between">
            {/* Left: Back + Tabs + Title */}
            <div className="flex items-center gap-4">
              <button onClick={() => window.history.back()}>←</button>
              <div>
                <div className="flex gap-2 mb-1">
                  <button className="px-3 py-1 bg-black text-white text-sm rounded">Video</button>
                  <button className="px-3 py-1 text-gray-600 text-sm rounded" disabled>Document</button>
                </div>
                <h1 className="text-xl font-semibold">{video.title}</h1>
              </div>
            </div>

            {/* Right: Action Buttons */}
            <div className="flex gap-2">
              <Button variant="outline" disabled>
                <Edit className="w-4 h-4 mr-2" />
                Edit
              </Button>
              <Button variant="outline" disabled>
                Translate
              </Button>
              <Button variant="outline" disabled>
                <Trash2 className="w-4 h-4 mr-2" />
                Delete
              </Button>
              <Button
                className="bg-purple-600 hover:bg-purple-700"
                onClick={() => window.location.href = `/api/videos/${video.id}/download`}
              >
                <Download className="w-4 h-4 mr-2" />
                Export Video
              </Button>
            </div>
          </div>
        </div>
      </div>

      {/* Video Player */}
      <div className="max-w-7xl mx-auto px-8 py-12">
        <div className="bg-gradient-to-br from-cyan-400 to-blue-500 rounded-xl p-12">
          <VideoPlayer url={video.processed_video_url!} />
        </div>
      </div>
    </div>
  );
}
```

**Component:** `src/components/video/VideoPlayer.tsx`

```typescript
'use client';

import ReactPlayer from 'react-player';

interface VideoPlayerProps {
  url: string;
}

export function VideoPlayer({ url }: VideoPlayerProps) {
  return (
    <div className="w-full max-w-4xl mx-auto bg-white rounded-lg shadow-2xl overflow-hidden">
      <ReactPlayer
        url={url}
        controls
        width="100%"
        height="100%"
        config={{
          file: {
            attributes: {
              controlsList: 'nodownload',
            },
          },
        }}
      />
    </div>
  );
}
```

**Testing:**
- [ ] Processing page shows animation
- [ ] Polling works (checks status every 3 seconds)
- [ ] Redirects to result page when complete
- [ ] Shows error message if processing fails
- [ ] Result page displays video correctly
- [ ] Video plays in player
- [ ] Export button downloads video

---

### Phase 11: Deployment
**Time:** 3-4 hours
**Dependencies:** All phases complete

#### 11.1 Vercel Deployment

**Prerequisites:**
- Vercel account
- Domain (optional, Vercel provides free subdomain)

**Steps:**

1. **Install Vercel CLI:**
```bash
npm install -g vercel
```

2. **Login:**
```bash
vercel login
```

3. **Deploy from root directory:**
```bash
vercel
```

4. **Configure environment variables in Vercel dashboard:**
   - Go to project settings → Environment Variables
   - Add all variables from `.env.local`
   - **Important:** Add for Production, Preview, and Development

5. **Deploy to production:**
```bash
vercel --prod
```

6. **Configure domain (optional):**
   - Vercel dashboard → Domains
   - Add custom domain: `app.zapra.ai`
   - Update DNS records as instructed

#### 11.2 Railway Worker Deployment

**Already done in Phase 9**, but verify:

```bash
cd worker
railway up
railway domain
```

**Set environment variables on Railway:**
```bash
# Generate a secure random API key first
node -e "console.log(require('crypto').randomBytes(32).toString('hex'))"

# Set all environment variables on Railway
railway variables set SUPABASE_URL=your_supabase_url
railway variables set SUPABASE_SERVICE_KEY=your_service_key
railway variables set AWS_ACCESS_KEY_ID=your_aws_key
railway variables set AWS_SECRET_ACCESS_KEY=your_aws_secret
railway variables set AWS_REGION=us-east-1
railway variables set AWS_S3_BUCKET=zapra-videos
railway variables set OPENAI_API_KEY=your_openai_key
railway variables set ELEVENLABS_API_KEY=your_elevenlabs_key
railway variables set ELEVENLABS_VOICE_ID=your_voice_id
railway variables set WORKER_API_KEY=<the_random_key_generated_above>
```

**Update Vercel environment variables:**
```bash
# In Vercel dashboard, add/update:
WORKER_URL=https://your-worker.up.railway.app
WORKER_API_KEY=<same_key_as_railway>
```

**IMPORTANT:** The `WORKER_API_KEY` must be identical on both Railway and Vercel!

#### 11.3 AWS S3 Configuration

**Create S3 Bucket:**

```bash
aws s3 mb s3://zapra-videos --region us-east-1
```

**Configure CORS:**

Create `cors.json`:
```json
[
  {
    "AllowedHeaders": [
      "Content-Type",
      "Content-Length",
      "Content-MD5",
      "x-amz-acl"
    ],
    "AllowedMethods": ["GET", "PUT", "HEAD"],
    "AllowedOrigins": [
      "http://localhost:3000",
      "https://zapra.ai",
      "https://app.zapra.ai",
      "chrome-extension://*"
    ],
    "ExposeHeaders": ["ETag"],
    "MaxAgeSeconds": 3000
  }
]
```

Apply:
```bash
aws s3api put-bucket-cors --bucket zapra-videos --cors-configuration file://cors.json
```

**Security Notes:**
- Only allow necessary HTTP methods (GET, PUT, HEAD)
- Restrict origins to production domains + localhost + Chrome extension
- Limit allowed headers to those needed for uploads
- **CRITICAL - Extension CORS Security:**
  - **For MVP testing:** `chrome-extension://*` wildcard is acceptable during development
  - **MUST FIX before public launch:** After Chrome Web Store approval, immediately replace `chrome-extension://*` with your specific extension ID
  - Example: `chrome-extension://abcdefghijklmnopqrstuvwxyz123456`
  - Get your stable extension ID from: https://chrome.google.com/webstore/developer/dashboard
  - **Why this matters:** The wildcard `*` allows ALL Chrome extensions (including malicious ones) to upload to your S3 bucket
  - **Update steps:**
    1. Get extension ID from Chrome Web Store dashboard
    2. Update `cors.json` with specific ID
    3. Reapply: `aws s3api put-bucket-cors --bucket zapra-videos --cors-configuration file://cors.json`
  - **Timeline:** Update within 24 hours of Chrome Web Store approval

**Set S3 Bucket Policy (prevent public access):**

Create `bucket-policy.json`:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyPublicListBucket",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:ListBucket",
      "Resource": "arn:aws:s3:::zapra-videos"
    },
    {
      "Sid": "DenyInsecureTransport",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": [
        "arn:aws:s3:::zapra-videos",
        "arn:aws:s3:::zapra-videos/*"
      ],
      "Condition": {
        "Bool": {
          "aws:SecureTransport": "false"
        }
      }
    },
    {
      "Sid": "AllowPresignedUploads",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::YOUR_ACCOUNT_ID:root"
      },
      "Action": [
        "s3:PutObject",
        "s3:GetObject"
      ],
      "Resource": "arn:aws:s3:::zapra-videos/*"
    }
  ]
}
```

Apply:
```bash
aws s3api put-bucket-policy --bucket zapra-videos --policy file://bucket-policy.json
```

**Bucket Policy Explanation:**
1. **DenyPublicListBucket** - Prevents anyone from listing bucket contents (no directory browsing)
2. **DenyInsecureTransport** - Forces HTTPS only (no HTTP uploads/downloads)
3. **AllowPresignedUploads** - Only your AWS account can create presigned URLs

**Set lifecycle policy (retention rules for MVP):**

```json
{
  "Rules": [
    {
      "Id": "Delete original recordings after 30 days",
      "Status": "Enabled",
      "Filter": {
        "Prefix": "videos/"
      },
      "Tags": [
        {
          "Key": "Type",
          "Value": "original"
        }
      ],
      "Expiration": {
        "Days": 30
      }
    }
  ]
}
```

**Data Retention Policy (for Privacy Policy & Chrome Web Store compliance):**

1. **Original recordings** (raw WebM from extension):
   - Automatically deleted after **30 days** via S3 lifecycle policy
   - Users cannot manually keep originals (reduces storage costs)
   - Original audio/video is not accessible after processing completes

2. **Processed videos** (MP4 with AI voice):
   - Retained indefinitely until user manually deletes
   - Users can delete from dashboard at any time
   - When user deletes video, DB record marked as deleted and S3 file removed within 24 hours

3. **Transcripts**:
   - Stored in database alongside video metadata
   - Deleted when user deletes video
   - Not shared with third parties (only sent to OpenAI Whisper during processing)

4. **User data**:
   - Account info (email, name) stored in Supabase
   - Subscription history retained for billing/compliance
   - Users can request account deletion (GDPR compliance)

5. **Analytics/logs**:
   - Processing logs retained for 90 days (debugging)
   - No PII in logs (only video IDs, timestamps, error messages)

**Privacy Policy Summary (one-liner for MVP):**
"Original recordings are automatically deleted after 30 days. Processed videos are retained until you delete them. We use OpenAI Whisper and ElevenLabs to process your audio, but do not share your content with other third parties."

#### 11.4 Supabase Configuration

**For production:**

1. **Disable email confirmation:**
   - Supabase dashboard → Authentication → Settings
   - Email → "Confirm email" = OFF

2. **Configure Google OAuth:**
   - Authentication → Providers → Google
   - Add OAuth credentials from Google Cloud Console
   - Set redirect URL: `https://app.zapra.ai/api/auth/callback`

3. **Set up custom SMTP (optional):**
   - For password reset emails
   - Settings → Auth → SMTP Settings

4. **Enable RLS policies** (already done in Phase 2)

5. **Backup strategy:**
   - Supabase Pro plan includes daily backups
   - Or use `pg_dump` for manual backups

#### 11.5 Chrome Extension Publication

**Prerequisites:**
- Chrome Web Store Developer account ($5 one-time fee)
- Extension zip file
- Screenshots of extension in use
- Privacy policy (can be added later if Chrome Web Store doesn't block submission)

**Steps:**

1. **Create extension package:**
```bash
cd chrome-extension
zip -r zapra-extension.zip * -x "*.DS_Store"
```

2. **Create Chrome Web Store listing:**
   - Go to [Chrome Web Store Developer Dashboard](https://chrome.google.com/webstore/devconsole)
   - Click "New Item"
   - Upload `zapra-extension.zip`

3. **Fill in listing details:**
   - Name: Zapra Screen Recorder
   - Description: Record your screen with AI-powered voice replacement
   - Category: Productivity
   - Language: English
   - Screenshots: Upload 3-5 screenshots showing recording flow
   - Icon: 128x128px logo
   - Small tile: 440x280px promotional image

4. **Privacy policy:**
   - May be required by Chrome Web Store (check during submission)
   - If needed: Create simple page at `${APP_URL}/privacy`
   - Should include: What data is collected (videos, audio), how it's used, retention policy
   - **Note:** Can be added post-MVP if Chrome Web Store allows initial submission without it

5. **Set post-install URL:**
   - In developer dashboard
   - Set to: `${APP_URL}/extension-installed`

6. **Submit for review:**
   - Takes 1-3 business days
   - You'll receive email when approved

7. **After approval:**
   - Extension available at: `https://chrome.google.com/webstore/detail/{extension-id}`
   - Update your app to link to this URL

#### 11.6 Dodo Payments Production Setup

**Steps:**

1. **Switch from test to live mode** in Dodo dashboard

2. **Create production products:**
   - Pro plan: $49/month
   - Scale plan: $249/month

3. **Update webhook URL:**
   - Production: `https://app.zapra.ai/api/payments/webhook`

4. **Get production API keys:**
   - Update in Vercel environment variables

5. **Test production payment flow:**
   - Use real credit card (will charge, can refund)
   - Verify webhook fires correctly
   - Check subscription created in database

#### 11.7 Monitoring & Logs

**Vercel:**
- Dashboard → Logs (real-time function logs)
- Dashboard → Analytics (usage stats)

**Railway:**
- Dashboard → Deployments → Logs
- Monitor worker health and errors

**Supabase:**
- Dashboard → Table Editor (check data)
- Dashboard → Logs (database queries)
- Dashboard → API (API usage)

**AWS S3:**
- CloudWatch for storage metrics
- S3 bucket metrics (storage size, requests)

**Set up alerts:**
- Vercel: Email alerts for failed deployments
- Railway: Email alerts for crashes
- Supabase: Email alerts for high DB load
- Consider: Sentry for error tracking (optional)

**Testing Checklist:**
- [ ] App deploys to Vercel successfully
- [ ] Worker deploys to Railway successfully
- [ ] Custom domain works (if configured)
- [ ] All environment variables set correctly
- [ ] S3 bucket configured and accessible
- [ ] Supabase production database works
- [ ] Google OAuth works in production
- [ ] Extension submits to Chrome Web Store
- [ ] Dodo Payments works in live mode
- [ ] Webhook fires correctly
- [ ] End-to-end flow works: Signup → Pay → Record → Process → Download
- [ ] Monitoring and logs accessible

---

## Appendix

### A. Cost Estimates

**Monthly Costs (100 users, 10 videos each = 1000 videos/month):**

| Service | Cost |
|---------|------|
| Vercel Pro (recommended) | $20/month |
| Railway Worker | $5-10/month |
| Supabase Pro (optional, Free tier may suffice) | $25/month or $0 |
| AWS S3 Storage (500GB) | $11.50/month |
| AWS S3 Data Transfer (realistic: 200GB/month) | $18/month |
| OpenAI Whisper (5 min avg, 1000 videos) | $30/month |
| ElevenLabs (1000 videos) | $100/month |
| Domain (.ai domain) | $2-10/month |
| **Total (with Supabase Free)** | **~$185-195/month** |
| **Total (with Supabase Pro)** | **~$210-220/month** |

**Bandwidth Calculation Details:**
- Average processed video: 80MB (5 min @ 2.5 Mbps H.264)
- Downloads per video: 2.5x (testing, re-downloads, sharing)
- Total transfer: 1000 videos × 80MB × 2.5 = 200GB/month
- Cost: 200GB × $0.09/GB = $18/month
- **Note:** Can spike to $30-40/month with heavy usage

**Per-Video Processing Cost:** ~$0.13 (Whisper $0.03 + ElevenLabs $0.10)

**Break-even Analysis:**
- If 5 Pro users ($49 × 5 = $245/month) → Profitable
- If 1 Scale user ($249/month) → Profitable
- **Margins remain healthy even with realistic bandwidth costs**

### B. Timeline Summary

| Phase | Description | Time |
|-------|-------------|------|
| 1 | Project Setup | 3-4 hours |
| 2 | Database & Supabase | 2-3 hours |
| 3 | Authentication | 3-4 hours |
| 4 | Payments | 4-5 hours |
| 5 | Dashboard & UI | 4-5 hours |
| 6 | Recording Flow | 2-3 hours |
| 7 | Chrome Extension | 5-6 hours |
| 8 | Upload/Download API | 2-3 hours |
| 9 | Railway Worker | 6-8 hours |
| 10 | Video Pages | 3-4 hours |
| 11 | Deployment | 3-4 hours |
| **Subtotal (Dev Time)** | | **37-49 hours** |
| **Buffer (Debugging & Testing)** | +50% | **+19-25 hours** |
| **Realistic Total** | | **60-80 hours** |

**Realistic Timeline:**
- **1 developer:** 8-10 days (assuming 8-hour workdays)
- **2 developers:** 5-6 days working in parallel
- **Note:** Original estimate assumed perfect implementation with zero bugs. Realistic estimate includes debugging, testing, and fixes.

### C. Features for Later (Post-MVP)

**Phase 2 Enhancements:**
1. ✅ **Video trimming** (cut start/end)
2. ✅ **Video cropping** (select region)
3. ✅ **Script editing** (edit transcript → regenerate voice)
4. ✅ **Multiple AI voices** (let user choose voice style)
5. ✅ **Language selection** (record in different languages)
6. ✅ **AI avatar overlay** (HeyGen or D-ID integration)
7. ✅ **Translation** (translate video to multiple languages)
8. ✅ **Visual zoom/pan** (auto-zoom to highlight areas)
9. ✅ **Background music** (add music tracks)
10. ✅ **Visual elements** (add arrows, text overlays, etc.)
11. ✅ **Record full screen/window** (not just Chrome tab)
12. ✅ **Upload video feature** (process pre-recorded videos)
13. ✅ **Watermark** (optional "Made with Zapra" on free tier)
14. ✅ **Email notifications** (notify when processing done)
15. ✅ **Admin dashboard** (monitor users, usage, errors)
16. ✅ **Analytics** (track user behavior, video views)
17. ✅ **Team collaboration** (share videos, collaborate)
18. ✅ **API access** (for programmatic video processing)
19. ✅ **Webhook notifications** (notify external systems)
20. ✅ **Recording interruption recovery** (resume if browser closes)
21. ✅ **Privacy policy page** (if Chrome Web Store blocks without it, can be added quickly)

### D. Security Best Practices

**Implemented in MVP:**
- ✅ Row Level Security (RLS) on all tables
- ✅ Auth tokens for extension ↔ API communication
- ✅ Payment validates identity (no bots)
- ✅ Signed S3 URLs for secure downloads
- ✅ Webhook signature verification (Dodo)

**Additional Considerations:**
- Rate limiting (if bot problem emerges)
- Captcha on signup (if bot problem emerges)
- Content moderation (scan transcripts for abuse)
- DDoS protection (Cloudflare in front of Vercel)
- API rate limiting (prevent abuse)

### E. Browser Compatibility

**Supported:**
- ✅ Chrome (primary)
- ✅ Edge (Chromium)
- ✅ Brave
- ✅ Opera

**Not Supported:**
- ❌ Firefox (different extension API)
- ❌ Safari (no extension support for screen recording)

**Note:** Extension only works on Chromium-based browsers. This is documented in marketing materials and help docs.

### F. Troubleshooting Guide

**Common Issues:**

1. **FFmpeg not found in worker:**
   - Railway includes FFmpeg by default
   - If not, add to `nixpacks.toml`:
   ```toml
   [phases.setup]
   aptPkgs = ["ffmpeg"]
   ```

2. **S3 upload fails with 403:**
   - Check AWS credentials
   - Verify bucket permissions
   - Check CORS configuration

3. **Whisper API rate limit:**
   - OpenAI: 50 requests/minute
   - If hit, implement queue with delay

4. **ElevenLabs quota exceeded:**
   - Check account limits
   - Upgrade plan if needed
   - Implement error handling

5. **Extension not connecting to app:**
   - Check `host_permissions` in manifest
   - Verify app URL matches
   - Check browser console for errors

6. **Supabase RLS blocking requests:**
   - Use service role key in worker (bypasses RLS)
   - Check policy definitions
   - Test in Supabase SQL editor

7. **Video processing timeout:**
   - Check worker logs in Railway
   - Verify worker is running
   - Check network connectivity (worker → S3, APIs)

8. **Payment webhook not firing:**
   - Verify webhook URL in Dodo dashboard
   - Check webhook signature verification
   - Test with Dodo webhook testing tool

### G. Maintenance Tasks

**Weekly:**
- [ ] Review error logs (Vercel, Railway)
- [ ] Check failed videos in database
- [ ] Monitor API costs (OpenAI, ElevenLabs)
- [ ] Check S3 storage usage

**Monthly:**
- [ ] Review user feedback
- [ ] Update dependencies (npm update)
- [ ] Check for security updates
- [ ] Review and optimize costs
- [ ] Backup database (manual if not on Supabase Pro)

**Quarterly:**
- [ ] Review and update pricing
- [ ] Plan new features
- [ ] Performance optimization
- [ ] User surveys

---

## Summary

This implementation plan provides a comprehensive roadmap for building Zapra.ai MVP with:

✅ **Payment-first approach** (no free tier, payment validates identity)
✅ **No email verification** (reduces friction, payment is verification)
✅ **No rate limiting** (users can use their allocation however they want)
✅ **Scalable architecture** (Vercel + Railway + Supabase + S3)
✅ **Clear pricing** (Pro $49, Scale $249, configurable)
✅ **Extension authentication** (secure token handoff)
✅ **Retry logic** (3 attempts for failed processing)
✅ **Production-ready** (for ~100-200 concurrent users)

**Timeline:** 12-16 days (1-2 developers)
**Monthly Cost:** ~$171-205 (at 1000 videos/month)
**Break-even:** 5 Pro users or 1 Scale user

Follow the phases sequentially, test thoroughly at each stage, and you'll have a production-ready MVP ready to generate revenue from day one!
