# CTF Writeups Overview

<div class="overview-grid" id="writeups-grid">
    <!-- Content will be dynamically generated -->
    <div class="overview-loader">Loading writeups...</div>
</div>

<script>
document.addEventListener('DOMContentLoaded', function() {
    // Categories and their colors
    const categories = {
        'htb': '#9FEF00',     // HackTheBox green
        'pg': '#FF5555',      // PG red
        'thm': '#1E88E5',     // TryHackMe blue
        'other': '#FFCB6B'    // Other
    };

    // Function to get all writeup links from the navigation
    function getWriteups() {
        // Get all navigation links
        const allLinks = Array.from(document.querySelectorAll('.md-nav__link'));
        
        // Filter out links that aren't writeups (exclude index pages, etc.)
        const writeupLinks = allLinks.filter(link => {
    const href = link.getAttribute('href');
    return href &&
           (href.includes('/htb/') || href.includes('/pg/') || href.includes('/thm/'));
});

        
        return writeupLinks.map(link => {
            // Get the category from the URL path
            let category = 'other';
            const href = link.getAttribute('href');
            
            if (href.includes('/htb/')) category = 'htb';
            else if (href.includes('/pg/')) category = 'pg';
            else if (href.includes('/thm/')) category = 'thm';
            
            // Get the difficulty if available
            let difficulty = 'unknown';
            const parentNode = link.closest('.md-nav__item');
            if (parentNode) {
                const text = parentNode.textContent.toLowerCase();
                if (text.includes('easy')) difficulty = 'easy';
                else if (text.includes('medium')) difficulty = 'medium';
                else if (text.includes('hard')) difficulty = 'hard';
                else if (text.includes('insane')) difficulty = 'insane';
            }
            
            return {
                title: link.textContent.trim(),
                url: href,
                category: category,
                difficulty: difficulty
            };
        });
    }

    // Function to create the grid of writeup cards
    function createWriteupsGrid(writeups) {
        const grid = document.getElementById('writeups-grid');
        grid.innerHTML = ''; // Clear loading message
        
        // Sort writeups by category and then by title
        writeups.sort((a, b) => {
            if (a.category !== b.category) {
                return a.category.localeCompare(b.category);
            }
            return a.title.localeCompare(b.title);
        });
        
        // Group writeups by category
        const categorized = {};
        writeups.forEach(writeup => {
            if (!categorized[writeup.category]) {
                categorized[writeup.category] = [];
            }
            categorized[writeup.category].push(writeup);
        });
        
        // Create section for each category
        Object.keys(categorized).forEach(category => {
            // Add category header
            const categoryHeader = document.createElement('h2');
            categoryHeader.textContent = getCategoryFullName(category);
            categoryHeader.classList.add('category-header');
            categoryHeader.style.color = categories[category] || '#FFFFFF';
            grid.appendChild(categoryHeader);
            
            // Create container for this category's cards
            const categoryContainer = document.createElement('div');
            categoryContainer.classList.add('category-container');
            grid.appendChild(categoryContainer);
            
            // Add cards for each writeup in this category
            categorized[category].forEach(writeup => {
                const card = createWriteupCard(writeup);
                categoryContainer.appendChild(card);
            });
        });
    }
    
    // Function to create a single writeup card
    function createWriteupCard(writeup) {
        const card = document.createElement('div');
        card.classList.add('writeup-card');
        card.dataset.difficulty = writeup.difficulty;
        
        // Add a colored border based on category
        card.style.borderColor = categories[writeup.category] || '#FFFFFF';
        
        // Add difficulty indicator if known
        if (writeup.difficulty !== 'unknown') {
            const difficultyIndicator = document.createElement('div');
            difficultyIndicator.classList.add('difficulty-indicator', writeup.difficulty);
            difficultyIndicator.textContent = writeup.difficulty.charAt(0).toUpperCase();
            card.appendChild(difficultyIndicator);
        }
        
        // Add writeup title
        const title = document.createElement('h3');
        title.textContent = writeup.title;
        card.appendChild(title);
        
        // Make the whole card clickable
        card.addEventListener('click', function() {
            window.location.href = writeup.url;
        });
        
        return card;
    }
    
    // Function to get full name of category
    function getCategoryFullName(category) {
        switch(category) {
            case 'htb': return 'HackTheBox';
            case 'pg': return 'Proving Grounds';
            case 'thm': return 'TryHackMe';
            default: return 'Other Writeups';
        }
    }
    
    // Initialize the grid
    const writeups = getWriteups();
    createWriteupsGrid(writeups);
});
</script>

<style>
/* Grid container */
.overview-grid {
    display: flex;
    flex-direction: column;
    gap: 2rem;
    margin-top: 2rem;
}

/* Category containers */
.category-container {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(250px, 1fr));
    gap: 1.5rem;
    margin-bottom: 2rem;
}

/* Category headers */
.category-header {
    margin-bottom: 1rem;
    padding-bottom: 0.5rem;
    border-bottom: 1px solid var(--terminal-green-dark);
    font-size: 1.5rem;
}

/* Individual writeup cards */
.writeup-card {
    background-color: #111111;
    border-left: 4px solid;
    padding: 1rem;
    border-radius: 0 4px 4px 0;
    cursor: pointer;
    position: relative;
    overflow: hidden;
    transition: transform 0.2s, box-shadow 0.2s;
    height: 100px;
    display: flex;
    align-items: center;
    justify-content: center;
}

.writeup-card:hover {
    transform: translateY(-3px);
    box-shadow: 0 5px 15px rgba(0, 255, 0, 0.1);
    background-color: #1a1a1a;
}

.writeup-card h3 {
    margin: 0;
    font-size: 1.1rem;
    text-align: center;
}

/* Difficulty indicators */
.difficulty-indicator {
    position: absolute;
    top: 0;
    right: 0;
    width: 24px;
    height: 24px;
    display: flex;
    align-items: center;
    justify-content: center;
    color: #000000;
    font-weight: bold;
    font-size: 0.8rem;
}

.difficulty-indicator.easy {
    background-color: #4CAF50; /* Green */
}

.difficulty-indicator.medium {
    background-color: #FFC107; /* Yellow */
}

.difficulty-indicator.hard {
    background-color: #F44336; /* Red */
}

.difficulty-indicator.insane {
    background-color: #9C27B0; /* Purple */
}

/* Loading spinner */
.overview-loader {
    text-align: center;
    font-style: italic;
    color: var(--terminal-green-dim);
    animation: pulse 1.5s infinite alternate;
}

@keyframes pulse {
    from { opacity: 0.6; }
    to { opacity: 1; }
}
</style>