LaTeX Notebook

A browser-based text and LaTeX editor with a live rendered preview, multiple notebooks, cloud synchronization, and high-resolution document export.

"Open Live Demo" (https://spacenebulae.github.io/latexnotebook/)

---

Overview

LaTeX Notebook is a lightweight, browser-based writing environment designed for combining ordinary text with mathematical notation.

The application uses a split-pane interface, allowing you to write text and LaTeX expressions in an editor while simultaneously viewing their rendered output in a separate preview pane.

Rather than requiring a dedicated LaTeX environment or a full compilation process, mathematical expressions are rendered directly in the browser using KaTeX.

The application also includes account authentication, cloud synchronization, multiple notebooks, preview navigation, and configurable export options.

Features

1. Live LaTeX Rendering

Write ordinary text alongside mathematical expressions using standard LaTeX delimiters.

- "$...$" for inline mathematical expressions.
- "$$...$$" for display-style mathematical expressions.
- Real-time rendering through "KaTeX" (https://katex.org/).
- Toggleable live rendering to temporarily pause preview updates.
- On-demand rendering when live rendering is disabled.
- Incremental preview updates to avoid unnecessarily rebuilding the entire document.
- Cached mathematical expressions to reduce redundant rendering operations.

Example:

The quadratic formula is

$$
x = \frac{-b \pm \sqrt{b^2 - 4ac}}{2a}
$$

For a simple function, we can write
$f(x) = x^2 + 2x + 1$.

2. LaTeX Autocomplete

The editor includes an extensive command autocomplete system designed to make writing mathematical expressions faster.

When typing a backslash inside math mode, matching commands appear automatically.

Features include:

- Gray ghost-text suggestions.
- Command previews with descriptions and mathematical symbols.
- Tab completion.
- Arrow-key navigation.
- Command matching and ranking.
- Usage-based suggestion prioritization.
- Automatic insertion of common mathematical structures.

The built-in command collection includes:

Category| Examples
Fractions and structures| "\frac", "\sqrt", "\binom"
Greek letters| "\alpha", "\theta", "\Omega"
Calculus| "\int", "\sum", "\lim", "\nabla"
Set theory| "\cup", "\cap", "\subseteq"
Logic| "\forall", "\exists", "\implies"
Relations| "\leq", "\neq", "\approx"
Operators| "\times", "\cdot", "\otimes"
Functions| "\sin", "\cos", "\log", "\exp"
Matrices| "\begin{pmatrix}", "\begin{bmatrix}"

Autocomplete suggestions are not a guarantee of rendering compatibility. Actual mathematical rendering is handled by KaTeX.

3. Multiple Notebooks

Organize different documents without having to create separate accounts or browser sessions.

Each account can maintain multiple notebooks, with support for:

- Creating new notebooks.
- Switching between notebooks.
- Renaming existing notebooks.
- Deleting notebooks.
- Automatically saving individual notebook contents.

All notebooks belonging to an account are stored together in its serialized document.

4. Account Authentication

User accounts are managed through "Supabase Authentication" (https://supabase.com/docs/guides/auth).

Available functionality includes:

- Email and password registration.
- Email and password sign-in.
- Sign-out functionality.
- Session restoration.
- Email confirmation support.
- Automatic loading of the authenticated user's notebooks.

Each user's document is associated with their unique Supabase authentication ID.

5. Cloud Synchronization

Notebook data is synchronized with a Supabase PostgreSQL database.

The synchronization system includes:

- Automatic cloud saving.
- Local browser snapshots.
- Offline editing support.
- Automatic synchronization retries.
- Session token refresh and authentication error recovery.
- Cloud update checks when returning to the application.
- Local-to-cloud reconciliation.
- Conflict detection when local and cloud versions differ.

The application maintains local copies of notebook data so that temporary connectivity problems do not immediately prevent continued editing.

When connectivity returns, the application attempts to reconcile local changes with the cloud document.

Conflicting changes are detected rather than silently overwritten.

6. Preview Zoom and Navigation

The rendered document has an independent navigation system.

Available controls include:

- Zoom in and out.
- Reset zoom to 100%.
- Drag-to-pan functionality.
- Spacebar + drag panning.
- Middle mouse button + drag panning.
- Custom horizontal and vertical preview scrollbars.
- Zoom-aware pan boundaries.

The preview can therefore be navigated independently of the editor, including when working with particularly large documents.

7. High-Resolution PNG Export

Export the entire rendered preview as a single PNG image.

The export system includes:

- Full-document capture.
- High-resolution rendering.
- A default export scale of up to 3×.
- Canvas dimension and pixel-count safeguards.
- KaTeX styling preservation.
- Foreign-object rendering with an HTML2Canvas fallback.

The application attempts to preserve the rendered appearance of the document rather than simply capturing the visible portion of the preview.

Keyboard shortcut: "Ctrl + S" or "Cmd + S"

8. Configurable PDF Export

The PDF export system provides several configuration options before generating a document.

Setting| Available options
Page size| Letter, A4, Legal, A5
Orientation| Portrait, Landscape
Margins| None, Narrow, Normal, Wide
Theme| Light, Current theme
Text size| 12 px, 15 px, 18 px
Quality| Standard (2×), High (3×)
Page numbers| Enabled, Disabled

Additional options include:

- Custom output filename.
- Optional document title.
- Saved export preferences.
- Automatic page dimension calculations.
- Smart pagination.
- Attempts to keep indivisible content together.
- Page numbering in the footer.

The pagination system analyzes rendered content to avoid unnecessarily splitting individual lines and other indivisible elements across pages.

---

Getting Started

Using the Live Version

The easiest way to use LaTeX Notebook is through its GitHub Pages deployment:

"https://spacenebulae.github.io/latexnotebook/" (https://spacenebulae.github.io/latexnotebook/)

Create an account or sign in to access the editor.

Running Locally

The application is primarily contained in a single HTML file, so no build system or package installation is required.

1. Clone the repository

git clone https://github.com/spacenebulae/latexnotebook.git

2. Navigate into the project directory

cd latexnotebook

3. Start a local web server

For example, using Python:

python -m http.server 8000

4. Open the application

Navigate to:

http://localhost:8000

A local web server is recommended instead of opening "index.html" directly, particularly for authentication and redirect functionality.

---

Supabase Configuration

LaTeX Notebook uses Supabase for authentication and cloud persistence.

A Supabase project must be configured with the appropriate database table and security policies.

1. Create the Documents Table

Execute the following SQL in the Supabase SQL Editor:

create table public.documents (
    user_id uuid primary key
        references auth.users (id)
        on delete cascade,

    content text not null default '',

    updated_at timestamptz not null default now()
);

alter table public.documents
enable row level security;

2. Configure Row-Level Security

Each authenticated user should only be able to access their own document.

Create the following policies:

create policy "Users can read their own document"
on public.documents
for select
to authenticated
using ((select auth.uid()) = user_id);


create policy "Users can insert their own document"
on public.documents
for insert
to authenticated
with check ((select auth.uid()) = user_id);


create policy "Users can update their own document"
on public.documents
for update
to authenticated
using ((select auth.uid()) = user_id)
with check ((select auth.uid()) = user_id);

Row-Level Security is essential because the application uses a browser-based Supabase client.

These policies ensure that users cannot access or modify another user's document through the database API.

3. Configure Supabase Credentials

The Supabase project URL and publishable key are defined near the beginning of the application script in "index.html".

const SUPABASE_URL = 'YOUR_SUPABASE_PROJECT_URL';

const SUPABASE_KEY = 'YOUR_SUPABASE_PUBLISHABLE_KEY';

Replace these values with the credentials from your Supabase project.

Important: Never expose a Supabase secret key or service-role key in client-side code. A publishable key is suitable for browser applications when proper database security policies are configured.

4. Configure Authentication URLs

In the Supabase dashboard, configure the authentication site's URL and allowed redirect URLs.

Include the application's deployed address and any local development addresses you intend to use.

For example:

https://spacenebulae.github.io/latexnotebook/
http://localhost:8000

If email confirmation is enabled, newly registered users may need to confirm their email addresses before signing in.

---

Technologies Used

Technology| Purpose
HTML5| Application structure
CSS3| Interface styling and layout
Vanilla JavaScript| Application logic and interactions
KaTeX 0.19.0| Mathematical expression rendering
Supabase| Authentication and cloud persistence
PostgreSQL| Notebook document storage
HTML2Canvas| HTML-to-canvas rendering
jsPDF| PDF generation
Google Fonts| Typography

External Dependencies

- "KaTeX" (https://katex.org/)
- "Supabase JavaScript Client" (https://supabase.com/docs/reference/javascript/introduction)
- "HTML2Canvas" (https://html2canvas.hertzen.com/)
- "jsPDF" (https://github.com/parallax/jsPDF)
- "Google Fonts" (https://fonts.google.com/)

Most external dependencies are loaded through CDNs. Export-related libraries are loaded when needed.

---

Browser Storage

The application uses browser "localStorage" to maintain local document snapshots, synchronization metadata, and user preferences.

Stored information includes:

- Local notebook documents.
- Synchronization state and timestamps.
- Previously synchronized document content.
- Conflict recovery backups.
- Editor theme preferences.
- Selected editor font.
- Live rendering preference.
- PDF export preferences.

Clearing browser site data can remove locally stored documents and preferences.

Cloud-synchronized documents remain stored separately in Supabase.

---

Keyboard and Mouse Controls

Input| Function
"Tab"| Accept autocomplete suggestion
"Arrow Up" / "Arrow Down"| Navigate autocomplete suggestions
"Escape"| Dismiss autocomplete
"Ctrl + S" / "Cmd + S"| Export preview as PNG
"Space + Drag"| Pan preview
"Middle Mouse + Drag"| Pan preview

---

Limitations and Notes

- The application is a browser-based editor and renderer, not a complete LaTeX compilation environment.
- KaTeX supports a defined subset of LaTeX functionality.
- Full ".tex" project compilation and external TeX package management are not implemented.
- An internet connection is required for authentication, CDN dependencies, and cloud synchronization.
- Local browser storage provides additional persistence but should not be considered a replacement for independent backups.
- Export quality and maximum document dimensions are ultimately subject to browser canvas limitations.

---

Repository

GitHub: "spacenebulae/latexnotebook" (https://github.com/spacenebulae/latexnotebook)

Live Application: "LaTeX Notebook" (https://spacenebulae.github.io/latexnotebook/)

---

LaTeX Notebook is an ongoing project focused on making mathematical writing, document organization, and rendered equation export accessible through a simple browser-based interface.
