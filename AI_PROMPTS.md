# Generative AI Disclosure

Using AI tools for this challenge is allowed. Be honest here — an accurate
"I used AI heavily" is worth more to us than an inaccurate "I did not".

**Did you use AI tools?** (yes / no): yes

**Which ones?** (e.g. ChatGPT, Claude, Copilot, Cursor): Antigravity (Gemini)

---

## 1. Key prompts

List the main prompts you used. Paraphrasing is fine.

1. `Can you help me understand the 3-container setup and the rules in the README?`
2. `Why is Docker failing on apt-get install with an error about nginx.conf conffile prompt?`
3. `Why did 127.0.0.1 fail for the backend, and how does Docker DNS resolve container names like "backend"?`
4. `How can I shrink the proxy image using Alpine and make Nginx run without root?`

---

## 2. Where the AI got it wrong

Describe **one concrete case** where the AI's answer was wrong, outdated, or
sub-optimal. Cover: what it suggested, how you noticed it was wrong, and what
you did instead. 3–5 sentences.

> When ./scripts/run-tests.sh gave a permission denied error, the AI suggested editing .github/workflows/ci.yml to run "bash ./scripts/run-tests.sh" instead of "./scripts/run-tests.sh". I realized that only masked the real issue because the script itself was missing its executable bit in the Git index. Instead of modifying the CI workflow, I made the file executable with chmod +x and used "git update-index --chmod=+x scripts/run-tests.sh". That way the executable permission is tracked in Git and works cleanly for anyone who clones the repo.

---

## 3. Anything in your submission you could not fully explain?

We would rather know. Write "nothing" if that is the case.

> I know Docker Compose uses internal DNS to resolve "backend" to the container IP. But I didn't fully dive into how Nginx handles DNS caching for upstreams if IPs change dynamically, so that's something I plan to learn more.
