# Tape Deck

Your own private music player: upload local audio files, build playlists, switch themes. No ads, no account, no server — everything runs in the browser, and it remembers your library and exact playback position using your browser's own storage.

## Get it live on a real domain (free hosting + your own domain)

### 1. Buy a domain (skip if you already have one)
Any registrar works — Namecheap, Porkbun, or Google Domains (now via Squarespace) are common, usually $10–15/year.

### 2. Put this code on GitHub
1. Create a free account at github.com if you don't have one.
2. Create a new repository (e.g. `tape-deck`).
3. Upload this whole folder to it (GitHub's web uploader works fine — drag the files in — or use `git push` if you're comfortable with git).

### 3. Deploy to Vercel (free)
1. Go to vercel.com and sign up (you can sign in with your GitHub account).
2. Click **Add New → Project**, and select the `tape-deck` repo you just created.
3. Vercel auto-detects it's a Vite project — leave the defaults and click **Deploy**.
4. In a minute or two, you'll get a live link like `tape-deck-yourname.vercel.app`. That's already a real, working website.

*(Netlify works the same way if you'd rather use that instead.)*

### 4. Connect your domain
1. In your Vercel project, go to **Settings → Domains** and enter your domain (e.g. `mytapedeck.com`).
2. Vercel shows you one or two DNS records to add.
3. Go to your domain registrar, find DNS settings, and add those records exactly as shown.
4. Wait 10 minutes to a few hours for DNS to update — then your domain loads the site directly.

## Running it locally first (optional, to test before deploying)
If you have Node.js installed:
```
npm install
npm run dev
```
Then open the local address it prints in your browser.

## What's real here
- 100% client-side: your audio files never touch a server, ever.
- Library, playlists, and exact playback position are saved in your browser's local storage, so they survive closing the tab — but they're tied to that one browser on that one device, not a cloud account. Clearing your browser data will clear them too.
- No backend, no database, no login — which also means there's no way to access your library from a different device or browser without re-uploading the same files there.
