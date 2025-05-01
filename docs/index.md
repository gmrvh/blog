<h1>CTF Writeups & Security Research</h1>
<p>This repository is intended to be reference for various cybersecurity challenges, methodologies, and tools</p>
<blockquote>
  <p>⚠️ <strong>Note:</strong> Writeups for <em>active</em> machines are <strong>censored</strong> in accordance with platform rules. Content is only published once the machine has been officially retired.</p>
</blockquote>
<hr>
<h2>Platform Collections</h2>
<p>Writeups are categorized by their respective cybersecurity platforms:</p>
<ul>
  <li><strong>HackTheBox (HTB)</strong><br>
  Solutions and technical breakdowns for retired machines and selected challenges.</li>
  <li><strong>Proving Grounds (PG)</strong><br>
  Step-by-step walkthroughs for completed machines.</li>
</ul>
<hr>
<div class="button-container">
    <a href="/ctf-writeups/web/" class="machine-overview-button">Machine Overview</a>
</div>

<style>
/* Button container */
.button-container {
    display: flex;
    justify-content: center;
    margin: 2rem 0;
    width: 100%;
}

/* Machine overview button */
.machine-overview-button {
    display: inline-block;
    background-color: #000000;
    color: #00ff00;
    padding: 0.8rem 1.5rem;
    border-radius: 4px;
    font-weight: bold;
    text-decoration: none;
    transition: all 0.3s ease;
    border: 2px solid #00ff00;
    text-align: center;
    font-size: 1.1rem;
    letter-spacing: 0.5px;
    text-transform: uppercase;
}

.machine-overview-button:hover {
    background-color: #001a00;
    box-shadow: 0 0 15px rgba(0, 255, 0, 0.5);
    transform: translateY(-3px);
}

.machine-overview-button:active {
    transform: translateY(0);
    box-shadow: 0 0 8px rgba(0, 255, 0, 0.3);
}

/* Pulsing animation for extra attention */
@keyframes pulse {
    0% { box-shadow: 0 0 0 0 rgba(0, 255, 0, 0.4); }
    70% { box-shadow: 0 0 0 10px rgba(0, 255, 0, 0); }
    100% { box-shadow: 0 0 0 0 rgba(0, 255, 0, 0); }
}

/* Apply pulsing animation when the page loads */
.machine-overview-button {
    animation: pulse 2s infinite;
}
</style>