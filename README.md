# onePaceMagnetExtractor [Broken]

With the origonal OnePace site now gone this tool is dead and will be archived as of today.
-------------------------------------------------------------------------------------------







A script to extract magnet links from the OnePace site.

## Overview

This script allows users to easily gather magnet links from the [OnePace website](https://onepace.net/watch) without manually searching for each episode's link.

## Requirements

- A modern web browser with developer tools (e.g. Google Chrome, Mozilla Firefox)

## Usage

1. Visit the [OnePace site](https://onepace.net/watch).
2. Navigate to "Watch One Pace Episodes".
3. Right-click anywhere on the page and select `Inspect` or `Inspect Element` from the context menu to open the developer console.
4. In the developer console, switch to the `Console` tab.
5. Copy the entire code block from this repository and paste it into the console.
6. Hit `Enter` to run the script.
7. Follow the on-screen prompts to extract magnet links for your desired arc.
8. You can also type `all` to extract all arc magnet links.
9. You will be prompted to download extracted data in `.txt` form.

## Video Guide
https://github.com/roshank231/onePaceMagnetExtractor/assets/83808600/d3d552c7-5fdb-4028-bb45-420210492652

## Code

Here's the script you'll need:

```javascript
// Run in the console of the OnePace site at https://onepace.net/watch
// Ensure that no episodes are already open.

let arcsData = [];
let arcSections = document.querySelectorAll('section[class^="Carousel_root__"]');
let cumulativeEpisodes = 0;

arcSections.forEach(section => {
    let arcName = section.querySelector('h2 > span > div').innerText;
    let episodeElements = section.querySelectorAll('[class^="CarouselSliderItem_item__"]');
    let episodesCount = episodeElements.length;

    arcsData.push({
        name: arcName,
        start: cumulativeEpisodes + 1,
        end: cumulativeEpisodes + episodesCount
    });

    cumulativeEpisodes += episodesCount;
});

function findArcByName(input) {
    for (let arc of arcsData) {
        if (arc.name.toLowerCase().includes(input.toLowerCase())) {
            return arc;
        }
    }
    return null;  // if no arc matches the input
}

function switchEpisode(episodeNumber, scrollToEpisode = true) {
    let episodeItems = document.querySelectorAll('[class^="CarouselSliderItem_item__"]');

    if (episodeNumber > episodeItems.length || episodeNumber < 1) {
        console.error("Invalid episode number");
        return;
    }

    let selectedEpisode = episodeItems[episodeNumber - 1];

    // Scroll the selected episode into view only if scrollToEpisode is true
    if (scrollToEpisode) {
        selectedEpisode.scrollIntoView({ behavior: 'smooth', block: 'center' });
    }

    selectedEpisode.click();
}

function getMagnetLink() {
    let links = Array.from(document.querySelectorAll('[class^="Carousel_container__"] [class^="Carousel_buttons__"] [href^="https://api.onepace.net/download/magnet.php"]')).map(a => a.href);

    return links;
}

function gatherMagnetLinks(arc) {
    console.log("Scraping Magnet Links for: " + arc.name + '\n' + "...");

    let complete = [];
    let intervalTime = 200;  // 200ms or 0.20 seconds -- EDIT THIS DELAY TO BE HIGHER IS HAVING ISSUES!!!

    let episode = arc.start;
    switchEpisode(episode);  // Scroll to the first episode

    return new Promise((resolve, reject) => {
        let interval = setInterval(() => {
            let links = getMagnetLink();
            
            links.forEach(link => {
                if (!complete.includes(link)) {
                    complete.push(link);
                }
            });
    
            if (complete.length > 0 || episode >= arc.end) {
                episode++;
    
                if (episode <= arc.end) {
                    switchEpisode(episode, false);
                } else {
                    clearInterval(interval);
    
                    // Deselect the last episode
                    let lastEpisode = document.querySelectorAll('[class^="CarouselSliderItem_item__"]')[episode - 2];
                    if (lastEpisode) lastEpisode.click();
    
                    // Scroll back to top
                    window.scrollTo({ top: 0, behavior: 'smooth' });
    
                    resolve(complete);
                }
            }
        }, intervalTime);
    });
}

function tellResult(result) {
    console.log("Completed gathering links. Here they are:");
    Object.keys(result).forEach( key => {
        if( key.toLowerCase() !== 'arc' ) {
            console.log( `${key} Arc - [${result[key].length}] unique magnet link(s).`);
            console.log(result[key].join('\n') );
        }
    });
    console.log("Right click and 'Save As' the above links in the console to save as a TXT or LOG file to easily paste elsewhere!");
    console.log("If the Links detected is less than expected, some/all episodes may already be batched into one link.");
} 

function gatherAllMagnetLinks() {
    const mapSeries = async (iterable, fn) => {
        const results = {};
      
        for (const x of iterable) {
            if( x.name.toLowerCase() !== 'arcs' )
                results[x.name] = await fn(x);
        }
      
        return results
    }
    
    mapSeries( arcsData, gatherMagnetLinks).then( res => {
        setTimeout(() => {
            tellResult(res);
            if( confirm('Save to file?') ) {
                download(res);
            }
        }, 1000); // This delay ensures that the previous tasks complete before showing the prompt.
    });
}

function download(data, fname) {
    if( !fname ) fname = 'OnePace Magnet Links';

    let strData = '';
    Object.keys(data).forEach( key => {
        strData += `~~ Arc: ${key} - ${data[key].length} link(s). ~~\n   ${data[key].join('\n   ')}\n\n`;
    });

    var file = new Blob([strData], {type: 'txt'});

    if (window.navigator.msSaveOrOpenBlob) // IE10+
        window.navigator.msSaveOrOpenBlob(file, fname);
    else { // Others
        var a = document.createElement("a");
        var url = URL.createObjectURL(file);

        a.href = url;
        a.download = fname;
        document.body.appendChild(a);
        a.click();
        setTimeout(function() {
            document.body.removeChild(a);
            window.URL.revokeObjectURL(url);  
        }, 0); 
    }
}

function initScrapingProcess() {
    // Gather user inputs
    let arcName = prompt("Enter the name of the arc:\n(You can also select 'all', will take a while.)");

    if( arcName.toLowerCase() === 'all' ) {
        gatherAllMagnetLinks();
    } else {
        let arc = findArcByName(arcName);
    
        if (arc) {
            gatherMagnetLinks(arc).then( res => {
                
                // Log Result
                setTimeout(() => {
                    tellResult({[arc.name]: res});

                    // Ask to save to file
                    if( confirm('Save to file?') ) {
                        download({[arc.name]: res}, `OnePace ${arc.name} Arc`);
                    }
                    
                    // Ask to rerun the process
                    if (confirm("Do you want to rerun the script to scrape another arc?")) {
                        initScrapingProcess();
                    }
                }, 1000); // This delay ensures that the previous tasks complete before showing the prompt.

            });
        } else if (confirm("Arc not found. Try again?")) {
            initScrapingProcess();
        }
    }
}

// Start the process
initScrapingProcess();
