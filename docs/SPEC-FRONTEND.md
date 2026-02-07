# Frontend Specification

## Overview

React + Vite admin dashboard for Field and Fen automation platform. Provides UI for:
1. Reviewing and approving files before publishing
2. Monitoring job status and pipeline health
3. Managing settings and API credentials
4. Viewing sales and analytics

**Design Aesthetic:** Meta/iOS-inspired (clean, minimal, professional)

---

## Tech Stack

| Layer | Technology | Why |
|-------|-----------|-----|
| Framework | React 18+ | Industry standard, great ecosystem |
| Build Tool | Vite | Fast dev server, modern bundler |
| State | Zustand | Simple, less boilerplate than Redux |
| Data Fetching | TanStack Query (React Query) | Caching, refetching, optimistic updates |
| GraphQL Client | graphql-request | Lightweight, works with React Query |
| Routing | React Router 6 | Standard routing solution |
| UI Components | Radix UI + Tailwind | Accessible primitives, utility-first CSS |
| Forms | React Hook Form | Performant, minimal re-renders |
| Validation | Zod | TypeScript-first schema validation |
| Icons | Lucide React | Consistent icon set |

---

## Project Structure

```
frontend/
├── src/
│   ├── main.tsx              # Entry point
│   ├── App.tsx               # Root component
│   ├── components/
│   │   ├── ui/               # Reusable UI components (Button, Input, etc)
│   │   │   ├── Button.tsx
│   │   │   ├── Input.tsx
│   │   │   ├── Card.tsx
│   │   │   └── ...
│   │   ├── layout/           # Layout components
│   │   │   ├── Sidebar.tsx
│   │   │   ├── Header.tsx
│   │   │   └── Layout.tsx
│   │   ├── files/            # File-related components
│   │   │   ├── FileCard.tsx
│   │   │   ├── FileList.tsx
│   │   │   └── FileReviewModal.tsx
│   │   ├── jobs/             # Job-related components
│   │   │   ├── JobsList.tsx
│   │   │   └── JobStatusBadge.tsx
│   │   └── ...
│   ├── pages/
│   │   ├── Dashboard.tsx
│   │   ├── Review.tsx
│   │   ├── Jobs.tsx
│   │   ├── Settings.tsx
│   │   └── ...
│   ├── lib/
│   │   ├── graphql.ts        # GraphQL client setup
│   │   ├── api.ts            # API utilities
│   │   └── utils.ts          # Helper functions
│   ├── hooks/
│   │   ├── useFiles.ts       # React Query hooks for files
│   │   ├── useJobs.ts
│   │   └── ...
│   ├── stores/
│   │   ├── authStore.ts      # Zustand store for auth
│   │   └── ...
│   ├── types/
│   │   ├── file.ts           # TypeScript types
│   │   ├── job.ts
│   │   └── ...
│   └── styles/
│       └── globals.css       # Global styles + Tailwind directives
├── public/
│   └── favicon.ico
├── index.html
├── package.json
├── tsconfig.json
├── tailwind.config.js
├── vite.config.ts
└── Dockerfile
```

---

## GraphQL Client Setup

**File:** `src/lib/graphql.ts`

```typescript
import { GraphQLClient } from 'graphql-request';

const endpoint = import.meta.env.VITE_API_URL || 'http://localhost:8000/graphql';

export const graphqlClient = new GraphQLClient(endpoint, {
  headers: {
    Authorization: `Bearer ${localStorage.getItem('token')}`,
  },
});

// Update token when it changes
export const setAuthToken = (token: string) => {
  localStorage.setItem('token', token);
  graphqlClient.setHeader('Authorization', `Bearer ${token}`);
};

export const clearAuthToken = () => {
  localStorage.removeItem('token');
  graphqlClient.setHeader('Authorization', '');
};
```

---

## Type Definitions

**File:** `src/types/file.ts`

```typescript
export type FileStatus = 
  | 'INGESTED'
  | 'PROCESSING'
  | 'PROCESSED'
  | 'APPROVED'
  | 'PUBLISHED'
  | 'FAILED';

export interface FileMetadata {
  title: string;
  description: string;
  hashtags: string[];
  captions: {
    instagram?: string;
    tiktok?: string;
    pinterest?: string;
  };
}

export interface File {
  id: string;
  originalFilename: string;
  processedFilename?: string;
  species: string;
  status: FileStatus;
  driveFileId?: string;
  metadata?: FileMetadata;
  width?: number;
  height?: number;
  aspectRatio?: string;
  createdAt: string;
  updatedAt: string;
  listings: Listing[];
  socialPosts: SocialPost[];
}

export interface Listing {
  id: string;
  fileId: string;
  platform: string;
  platformListingId: string;
  listingUrl?: string;
  status: string;
  createdAt: string;
}

export interface SocialPost {
  id: string;
  fileId: string;
  platform: string;
  postType?: string;
  platformPostId?: string;
  postUrl?: string;
  postedAt: string;
}
```

**File:** `src/types/job.ts`

```typescript
export type JobStatus = 'PENDING' | 'RUNNING' | 'COMPLETED' | 'FAILED';

export interface Job {
  id: string;
  fileId?: string;
  commandType: string;
  status: JobStatus;
  attempts: number;
  lastError?: string;
  createdAt: string;
  completedAt?: string;
}
```

---

## React Query Hooks

**File:** `src/hooks/useFiles.ts`

```typescript
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';
import { graphqlClient } from '../lib/graphql';
import { gql } from 'graphql-request';
import { File, FileStatus } from '../types/file';

const GET_FILES = gql`
  query GetFiles($status: FileStatus, $species: String, $limit: Int, $offset: Int) {
    files(status: $status, species: $species, limit: $limit, offset: $offset) {
      id
      originalFilename
      processedFilename
      species
      status
      metadata {
        title
        description
        hashtags
      }
      width
      height
      aspectRatio
      createdAt
      updatedAt
      listings {
        id
        platform
        listingUrl
      }
    }
  }
`;

const APPROVE_FILE = gql`
  mutation ApproveFile($id: ID!) {
    approveFile(id: $id) {
      id
      status
    }
  }
`;

export const useFiles = (filters: {
  status?: FileStatus;
  species?: string;
  limit?: number;
  offset?: number;
}) => {
  return useQuery<File[]>({
    queryKey: ['files', filters],
    queryFn: async () => {
      const data = await graphqlClient.request(GET_FILES, filters);
      return data.files;
    },
  });
};

export const useApproveFile = () => {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: async (fileId: string) => {
      const data = await graphqlClient.request(APPROVE_FILE, { id: fileId });
      return data.approveFile;
    },
    onSuccess: () => {
      // Invalidate and refetch files
      queryClient.invalidateQueries({ queryKey: ['files'] });
    },
  });
};
```

---

## Pages

### Dashboard (Overview)

**File:** `src/pages/Dashboard.tsx`

```tsx
import { useFiles } from '../hooks/useFiles';
import { useJobs } from '../hooks/useJobs';
import { Card } from '../components/ui/Card';

export const Dashboard = () => {
  const { data: files, isLoading: filesLoading } = useFiles({ limit: 10 });
  const { data: jobs, isLoading: jobsLoading } = useJobs({ limit: 10 });

  const stats = {
    totalFiles: files?.length || 0,
    processingFiles: files?.filter(f => f.status === 'PROCESSING').length || 0,
    pendingReview: files?.filter(f => f.status === 'PROCESSED').length || 0,
    publishedFiles: files?.filter(f => f.status === 'PUBLISHED').length || 0,
  };

  return (
    <div className="space-y-6">
      <h1 className="text-3xl font-bold">Dashboard</h1>

      {/* Stats Grid */}
      <div className="grid grid-cols-1 md:grid-cols-4 gap-4">
        <Card>
          <h3 className="text-sm font-medium text-gray-500">Total Files</h3>
          <p className="text-2xl font-bold">{stats.totalFiles}</p>
        </Card>
        <Card>
          <h3 className="text-sm font-medium text-gray-500">Processing</h3>
          <p className="text-2xl font-bold text-blue-600">{stats.processingFiles}</p>
        </Card>
        <Card>
          <h3 className="text-sm font-medium text-gray-500">Pending Review</h3>
          <p className="text-2xl font-bold text-yellow-600">{stats.pendingReview}</p>
        </Card>
        <Card>
          <h3 className="text-sm font-medium text-gray-500">Published</h3>
          <p className="text-2xl font-bold text-green-600">{stats.publishedFiles}</p>
        </Card>
      </div>

      {/* Recent Activity */}
      <Card>
        <h2 className="text-xl font-bold mb-4">Recent Activity</h2>
        {/* Job list */}
      </Card>
    </div>
  );
};
```

### Review Queue

**File:** `src/pages/Review.tsx`

```tsx
import { useState } from 'react';
import { useFiles, useApproveFile } from '../hooks/useFiles';
import { FileCard } from '../components/files/FileCard';
import { FileReviewModal } from '../components/files/FileReviewModal';
import { Button } from '../components/ui/Button';

export const Review = () => {
  const { data: files, isLoading } = useFiles({ status: 'PROCESSED' });
  const [selectedFile, setSelectedFile] = useState<File | null>(null);
  const approveFile = useApproveFile();

  const handleApprove = async (fileId: string) => {
    await approveFile.mutateAsync(fileId);
    setSelectedFile(null);
  };

  if (isLoading) return <div>Loading...</div>;

  return (
    <div className="space-y-6">
      <div className="flex justify-between items-center">
        <h1 className="text-3xl font-bold">Review Queue</h1>
        <p className="text-gray-500">{files?.length || 0} items pending</p>
      </div>

      <div className="grid grid-cols-1 md:grid-cols-3 gap-6">
        {files?.map((file) => (
          <FileCard
            key={file.id}
            file={file}
            onClick={() => setSelectedFile(file)}
          />
        ))}
      </div>

      {selectedFile && (
        <FileReviewModal
          file={selectedFile}
          onApprove={() => handleApprove(selectedFile.id)}
          onClose={() => setSelectedFile(null)}
        />
      )}
    </div>
  );
};
```

---

## Components

### FileCard

**File:** `src/components/files/FileCard.tsx`

```tsx
import { File } from '../../types/file';
import { Card } from '../ui/Card';
import { Badge } from '../ui/Badge';

interface FileCardProps {
  file: File;
  onClick?: () => void;
}

export const FileCard = ({ file, onClick }: FileCardProps) => {
  const statusColor = {
    INGESTED: 'gray',
    PROCESSING: 'blue',
    PROCESSED: 'yellow',
    APPROVED: 'green',
    PUBLISHED: 'green',
    FAILED: 'red',
  }[file.status];

  return (
    <Card className="cursor-pointer hover:shadow-lg transition-shadow" onClick={onClick}>
      {/* Image Preview */}
      <div className="aspect-w-16 aspect-h-9 bg-gray-100 rounded-t-lg">
        {file.driveFileId && (
          <img
            src={`https://drive.google.com/thumbnail?id=${file.driveFileId}&sz=w400`}
            alt={file.metadata?.title || file.originalFilename}
            className="object-cover w-full h-full rounded-t-lg"
          />
        )}
      </div>

      {/* Content */}
      <div className="p-4 space-y-2">
        <div className="flex justify-between items-start">
          <h3 className="font-semibold text-lg line-clamp-2">
            {file.metadata?.title || file.originalFilename}
          </h3>
          <Badge color={statusColor}>{file.status}</Badge>
        </div>

        <p className="text-sm text-gray-500">{file.species}</p>

        {file.metadata?.description && (
          <p className="text-sm text-gray-600 line-clamp-2">
            {file.metadata.description}
          </p>
        )}

        {/* Listings */}
        {file.listings.length > 0 && (
          <div className="flex gap-2 flex-wrap">
            {file.listings.map((listing) => (
              <Badge key={listing.id} variant="outline">
                {listing.platform}
              </Badge>
            ))}
          </div>
        )}
      </div>
    </Card>
  );
};
```

### FileReviewModal

**File:** `src/components/files/FileReviewModal.tsx`

```tsx
import { File } from '../../types/file';
import { Button } from '../ui/Button';
import { Modal } from '../ui/Modal';

interface FileReviewModalProps {
  file: File;
  onApprove: () => void;
  onClose: () => void;
}

export const FileReviewModal = ({ file, onApprove, onClose }: FileReviewModalProps) => {
  return (
    <Modal open onClose={onClose} title="Review File">
      <div className="space-y-4">
        {/* Large Image Preview */}
        <div className="w-full aspect-w-16 aspect-h-9 bg-gray-100 rounded-lg">
          {file.driveFileId && (
            <img
              src={`https://drive.google.com/uc?id=${file.driveFileId}`}
              alt={file.metadata?.title || file.originalFilename}
              className="object-contain w-full h-full rounded-lg"
            />
          )}
        </div>

        {/* Metadata */}
        <div>
          <h3 className="font-bold text-xl">{file.metadata?.title}</h3>
          <p className="text-gray-500">{file.species}</p>
        </div>

        {file.metadata?.description && (
          <div>
            <h4 className="font-semibold mb-2">Description</h4>
            <p className="text-sm text-gray-700">{file.metadata.description}</p>
          </div>
        )}

        {file.metadata?.hashtags && (
          <div>
            <h4 className="font-semibold mb-2">Hashtags</h4>
            <div className="flex flex-wrap gap-2">
              {file.metadata.hashtags.map((tag) => (
                <span key={tag} className="text-sm bg-gray-100 px-2 py-1 rounded">
                  #{tag}
                </span>
              ))}
            </div>
          </div>
        )}

        {/* Actions */}
        <div className="flex gap-3 justify-end pt-4">
          <Button variant="outline" onClick={onClose}>
            Cancel
          </Button>
          <Button onClick={onApprove}>
            Approve & Publish
          </Button>
        </div>
      </div>
    </Modal>
  );
};
```

---

## UI Components (Radix + Tailwind)

### Button

**File:** `src/components/ui/Button.tsx`

```tsx
import { ButtonHTMLAttributes, forwardRef } from 'react';
import { cn } from '../../lib/utils';

interface ButtonProps extends ButtonHTMLAttributes<HTMLButtonElement> {
  variant?: 'primary' | 'outline' | 'ghost';
  size?: 'sm' | 'md' | 'lg';
}

export const Button = forwardRef<HTMLButtonElement, ButtonProps>(
  ({ className, variant = 'primary', size = 'md', ...props }, ref) => {
    return (
      <button
        ref={ref}
        className={cn(
          'inline-flex items-center justify-center rounded-lg font-medium transition-colors',
          'focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-offset-2',
          'disabled:opacity-50 disabled:pointer-events-none',
          {
            'bg-blue-600 text-white hover:bg-blue-700 focus-visible:ring-blue-600':
              variant === 'primary',
            'border border-gray-300 bg-white text-gray-700 hover:bg-gray-50 focus-visible:ring-gray-300':
              variant === 'outline',
            'text-gray-700 hover:bg-gray-100 focus-visible:ring-gray-300':
              variant === 'ghost',
            'px-3 py-1.5 text-sm': size === 'sm',
            'px-4 py-2 text-base': size === 'md',
            'px-6 py-3 text-lg': size === 'lg',
          },
          className
        )}
        {...props}
      />
    );
  }
);
```

### Card

```tsx
import { HTMLAttributes, forwardRef } from 'react';
import { cn } from '../../lib/utils';

export const Card = forwardRef<HTMLDivElement, HTMLAttributes<HTMLDivElement>>(
  ({ className, ...props }, ref) => (
    <div
      ref={ref}
      className={cn(
        'rounded-lg border border-gray-200 bg-white shadow-sm',
        className
      )}
      {...props}
    />
  )
);
```

---

## Routing

**File:** `src/App.tsx`

```tsx
import { BrowserRouter, Routes, Route, Navigate } from 'react-router-dom';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { Layout } from './components/layout/Layout';
import { Dashboard } from './pages/Dashboard';
import { Review } from './pages/Review';
import { Jobs } from './pages/Jobs';
import { Settings } from './pages/Settings';

const queryClient = new QueryClient();

function App() {
  return (
    <QueryClientProvider client={queryClient}>
      <BrowserRouter>
        <Routes>
          <Route path="/" element={<Layout />}>
            <Route index element={<Navigate to="/dashboard" replace />} />
            <Route path="dashboard" element={<Dashboard />} />
            <Route path="review" element={<Review />} />
            <Route path="jobs" element={<Jobs />} />
            <Route path="settings" element={<Settings />} />
          </Route>
        </Routes>
      </BrowserRouter>
    </QueryClientProvider>
  );
}

export default App;
```

---

## Environment Variables

**File:** `.env`

```bash
VITE_API_URL=http://localhost:8000
```

**File:** `.env.production`

```bash
VITE_API_URL=https://api.fieldandfen.art
```

---

## Build & Deploy

### Development

```bash
npm install
npm run dev
```

### Production Build

```bash
npm run build
npm run preview  # Test production build locally
```

### Docker (see SPEC-DOCKER.md for full config)

```dockerfile
FROM node:20-alpine AS builder
WORKDIR /build
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM nginx:alpine
COPY --from=builder /build/dist /usr/share/nginx/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

---

## Testing

Use Vitest (compatible with Vite) + React Testing Library.

**File:** `src/components/FileCard.test.tsx`

```tsx
import { render, screen } from '@testing-library/react';
import { FileCard } from './FileCard';

describe('FileCard', () => {
  it('renders file title', () => {
    const file = {
      id: '1',
      originalFilename: 'test.jpg',
      species: 'whitetail-deer',
      status: 'PROCESSED' as const,
      metadata: {
        title: 'Test Buck',
        description: 'A test image',
        hashtags: [],
        captions: {},
      },
      listings: [],
      socialPosts: [],
      createdAt: '2026-01-01T00:00:00Z',
      updatedAt: '2026-01-01T00:00:00Z',
    };

    render(<FileCard file={file} />);

    expect(screen.getByText('Test Buck')).toBeInTheDocument();
    expect(screen.getByText('PROCESSED')).toBeInTheDocument();
  });
});
```

---

## Accessibility

- Use semantic HTML (`<button>`, `<nav>`, etc.)
- ARIA labels where needed
- Keyboard navigation (tab, enter, escape)
- Color contrast (WCAG AA minimum)
- Focus indicators

---

## Next Steps

After implementing this spec:
1. Set up Vite project with TypeScript
2. Install dependencies (React Query, Radix, Tailwind)
3. Build UI components (Button, Card, Modal, etc.)
4. Implement pages (Dashboard, Review, Jobs)
5. Connect to GraphQL API
6. Test in Docker
7. Deploy alongside backend
