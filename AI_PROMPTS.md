# Generative AI Disclosure

Using AI tools for this challenge is allowed. Be honest here — an accurate
"I used AI heavily" is worth more to us than an inaccurate "I did not".

**Did you use AI tools?** (yes / no): yes

**Which ones?** (e.g. ChatGPT, Claude, Copilot, Cursor): Antigravity (Gemini)

---

## 1. Key prompts

List the main prompts you used. Paraphrasing is fine.

1. `I got a home assignment from F5 company, research the folder and summarize what needs to be done, rules, and how to submit.`
2. `Explain what caused the apt-get conffile prompt failure during proxy image build and how to fix it.`
3. `The proxy connects to 127.0.0.1:8080 and fails with connection refused; explain container networking in Docker Compose.`
4. `The backend loads config keys with sleep delays; how do we coordinate Docker Compose startup without modifying backend/app.py?`
5. `Explain the difference between Ubuntu and Alpine base images and what changes are needed to run Nginx as a non-root user.`

---

## 2. Where the AI got it wrong

Describe **one concrete case** where the AI's answer was wrong, outdated, or
sub-optimal. Cover: what it suggested, how you noticed it was wrong, and what
you did instead. 3–5 sentences.

> When addressing the bonus questions, the AI initially proposed a single monolithic patch that simultaneously introduced `nginx:alpine`, non-root user permissions, temporary PID redirection, and extra security headers into both the Dockerfile and configuration. I noticed this mixed multiple distinct requirements into one large change instead of isolating the image size reduction first. I asked the AI to explain the changes line-by-line rather than dumping code, and to separate the minimal image migration from the non-root configuration. This allowed us to verify the image reduction (from 325MB to 68.5MB) before layering on non-root permissions.

---

## 3. Anything in your submission you could not fully explain?

We would rather know. Write "nothing" if that is the case.

> nothing
