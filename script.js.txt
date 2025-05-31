document.addEventListener('DOMContentLoaded', () => {
    const canvas = document.getElementById('neuralNetworkCanvas');
    const ctx = canvas.getContext('2d');

    let width, height;
    let nodes = [];
    let pulses = [];
    let projectCards = document.querySelectorAll('.project-card');
    

    // --- Adjustable Parameters ---
    const PARAMS = {
        // Genesis
        genesisDuration: 4000, // ms
        genesisMaxNodes: 500,
        genesisSeedNodes: 1,
        genesisNodeBudDistance: 300,
        genesisConnectionGrowthRate: 0.0008,
        genesisNodeGrowthRate: 0.005, // Chance per frame to add node during genesis

        minNodeDistance: 30, // Minimum distance between nodes to prevent overlap

        // Cognitive Load
        cognitiveLoadThreshold: 3, // Clicks within cognitiveLoadTimeWindow to trigger
        cognitiveLoadTimeWindow: 1500, // ms
        cognitiveLoadMaxLevel: 1.0,
        cognitiveLoadDecayRate: 0.005, // per frame
        cognitiveLoadPulseMultiplier: 2.5,
        cognitiveLoadBrightnessMultiplier: 1.8,
        cognitiveLoadColorShiftActive: true,
        cognitiveLoadPulseColor: 'rgba(255, 100, 100, 0.9)', // Reddish pulse during overload
        cognitiveLoadNodeColor: 'rgba(200, 150, 150, 0.9)', // Reddish node during overload

        // Focus Mode
        focusModeDimOpacity: 0.2, // Opacity for dimmed network elements (nodes/connections)
        
        // Node Realism
        nodeBaseRadiusMin: 1.2,
        nodeBaseRadiusMax: 2.2,
        nodeIrregularity: 0.4, // 0 = perfect circle, 1 = very irregular
        nodeCoronaFactor: 1.6,
        nodeCoronaBaseOpacity: 0.8,
        nodePulsingSpeed: 0.01, // For subtle breathing effect
        nodePulsingAmplitude: 0.2,

        // Connection Realism
        connectionCurvatureMax: 0.015, // Max normalized curvature
        connectionBaseWidth: 0.25,
        connectionTaperFactor: 0.05, // 0 = no taper, 1 = full taper to point
        connectionGlow: true,
        connectionGlowColor: 'rgba(70, 130, 180, 0.8)', // Steel blue glow
        connectionGlowWidthMultiplier: 3, // Multiplier of base width

        // Synaptic Junction
        synapticSparkDuration: 300, // ms
        synapticSparkBaseRadius: 0.8,
        synapticSparkRadiusMultiplier: 2.5,
        synapticSparkColor: 'rgba(255, 255, 220, 0.95)', // Bright yellowish spark

        // Core Network & Pulse Params
        connectionDistance: 160,
        connectionProbability: 0.35,
        pulseRadius: 1.1,
        minPulseSpeed: 0.5,
        maxPulseSpeed: 1.2,
        pulseSpawnRate: 0.006,
        basePulseColor: 'rgba(42, 82, 190, 0.85)',
        baseNodeColor: 'rgba(135, 144, 165, 0.95)',
        baseConnectionColor: 'rgb(16, 42, 94)',
        
        // Mouse Interaction
        mouseEffectRadius: 120,
        mouseNodeBrightnessFactor: 1.6,
        mousePulseSpawnOnClick: true, // Spawn pulses on canvas click
        mouseClickRipple: true,
        mouseClickRippleDuration: 500, //ms
        mouseClickRippleMaxRadius: 100,
        mouseClickRippleColor: 'rgba(150, 180, 255, 0.3)'
    };

    let mouse = { x: undefined, y: undefined, isDown: false };
    let genesisInProgress = true;
    let genesisStartTime = Date.now();
    let nodesCreatedDuringGenesis = 0;

    let cognitiveLoadLevel = 0;
    let userInteractionTimestamps = [];
    let lastInteractionTime = 0;

    let focusedProjectCard = null;
    let activeSynapticSparks = [];
    let activeClickRipples = [];
    let frameCount = 0; // For pulsing effects

    function getRandomInRange(min, max) {
        return Math.random() * (max - min) + min;
    }

    function resizeCanvas() {
        width = canvas.width = window.innerWidth;
        height = canvas.height = window.innerHeight;
        if (!genesisInProgress) { // Don't re-init full network if genesis happened
            // Adjust existing node positions to new canvas size (optional, or re-init)
            // For simplicity, let's re-init if not in genesis, or just let genesis run its course
             nodes = []; pulses = []; activeSynapticSparks = []; activeClickRipples = [];
             genesisInProgress = true;
             genesisStartTime = Date.now();
             nodesCreatedDuringGenesis = 0;
             initGenesisNetwork();
        } else {
             initGenesisNetwork(); // Restart genesis if it was interrupted by resize
        }
    }

    class Node {
        constructor(x, y) {
            this.x = x;
            this.y = y;
            this.baseRadius = getRandomInRange(PARAMS.nodeBaseRadiusMin, PARAMS.nodeBaseRadiusMax);
            this.currentRadius = this.baseRadius;
            this.irregularityPoints = [];
            this.connections = [];
            this.brightness = 1;
            this.opacity = 1; // For focus mode dimming
            this.id = Math.random().toString(36).substr(2, 9); // Unique ID

            // Generate points for irregular shape
            const numPoints = Math.floor(getRandomInRange(5, 8));
            for (let i = 0; i < numPoints; i++) {
                const angle = (i / numPoints) * Math.PI * 2;
                const radiusOffset = 1 - PARAMS.nodeIrregularity + Math.random() * PARAMS.nodeIrregularity * 2;
                this.irregularityPoints.push({
                    x: Math.cos(angle) * radiusOffset,
                    y: Math.sin(angle) * radiusOffset
                });
            }
        }

        draw(currentCognitiveLoad) {
            let effectiveRadius = this.currentRadius * this.brightness;
            effectiveRadius *= (1 + Math.sin(frameCount * PARAMS.nodePulsingSpeed + this.id.charCodeAt(0)) * PARAMS.nodePulsingAmplitude); // Subtle pulsing
            
            let nodeColor = PARAMS.baseNodeColor;
            if (currentCognitiveLoad > 0 && PARAMS.cognitiveLoadColorShiftActive) {
                const r = Math.min(255, 80 + 120 * currentCognitiveLoad);
                const g = Math.max(0, 88 - 88 * currentCognitiveLoad);
                const b = Math.max(0, 106 - 106 * currentCognitiveLoad);
                nodeColor = `rgba(${r},${g},${b},${0.75 + 0.2 * currentCognitiveLoad})`;
            }
            
            const finalOpacity = this.opacity * (nodeColor.match(/rgba?\([^,]+,[^,]+,[^,]+,([^)]+)\)/)?.[1] || 1);


            // Corona
            ctx.beginPath();
            ctx.arc(this.x, this.y, effectiveRadius * PARAMS.nodeCoronaFactor, 0, Math.PI * 2);
            const coronaOpacity = PARAMS.nodeCoronaBaseOpacity * this.brightness * this.opacity;
            const coronaGradient = ctx.createRadialGradient(this.x, this.y, effectiveRadius, this.x, this.y, effectiveRadius * PARAMS.nodeCoronaFactor);
            coronaGradient.addColorStop(0, `rgba(${nodeColor.match(/\d+/g).slice(0,3).join(',')}, 0)`);
            coronaGradient.addColorStop(1, `rgba(${nodeColor.match(/\d+/g).slice(0,3).join(',')}, ${coronaOpacity})`);
            ctx.fillStyle = coronaGradient;
            ctx.fill();
            
            // Core (irregular shape)
            ctx.beginPath();
            ctx.moveTo(this.x + this.irregularityPoints[0].x * effectiveRadius, this.y + this.irregularityPoints[0].y * effectiveRadius);
            for (let i = 1; i < this.irregularityPoints.length; i++) {
                ctx.lineTo(this.x + this.irregularityPoints[i].x * effectiveRadius, this.y + this.irregularityPoints[i].y * effectiveRadius);
            }
            ctx.closePath();
            ctx.fillStyle = nodeColor.replace(/rgba?\(([^)]+)\)/, `rgba($1, ${finalOpacity})`);
            ctx.fill();
        }

        connect(otherNode) {
            if (!this.connections.includes(otherNode.id) && this.connections.length < 6) { // Limit connections
                 this.connections.push(otherNode.id);
                 otherNode.connections.push(this.id);
            }
        }
    }
    
    function getNodeById(id) {
        return nodes.find(n => n.id === id);
    }

    class Pulse {
        constructor(startNode, endNode) {
            this.startNode = startNode;
            this.endNode = endNode;
            this.x = startNode.x;
            this.y = startNode.y;
            this.radius = PARAMS.pulseRadius;
            this.speed = getRandomInRange(PARAMS.minPulseSpeed, PARAMS.maxPulseSpeed);
            this.progress = 0; // 0 to 1
            this.opacity = 1; // For focus mode dimming
        }

        update(currentCognitiveLoad) {
            let effectiveSpeed = this.speed * (1 + currentCognitiveLoad * (PARAMS.cognitiveLoadPulseMultiplier -1));
            this.progress += effectiveSpeed / 100;

            if (this.progress >= 1) {
                // Trigger synaptic spark
                activeSynapticSparks.push({
                    node: this.endNode,
                    startTime: Date.now(),
                    radius: PARAMS.synapticSparkBaseRadius * (1 + Math.random() * (PARAMS.synapticSparkRadiusMultiplier -1))
                });
                return true; // Signal to remove
            }
            this.x = this.startNode.x + (this.endNode.x - this.startNode.x) * this.progress;
            this.y = this.startNode.y + (this.endNode.y - this.startNode.y) * this.progress;
            return false;
        }

        draw(currentCognitiveLoad) {
            let pulseColor = PARAMS.basePulseColor;
            if (currentCognitiveLoad > 0 && PARAMS.cognitiveLoadColorShiftActive) {
                 pulseColor = PARAMS.cognitiveLoadPulseColor;
            }
            const finalOpacity = this.opacity * (pulseColor.match(/rgba?\([^,]+,[^,]+,[^,]+,([^)]+)\)/)?.[1] || 1);

            ctx.beginPath();
            ctx.arc(this.x, this.y, this.radius, 0, Math.PI * 2);
            ctx.fillStyle = pulseColor.replace(/rgba?\(([^)]+)\)/, `rgba($1, ${finalOpacity})`);
            ctx.fill();
        }
    }

        function initGenesisNetwork() {
        nodes = [];
        pulses = [];
        activeSynapticSparks = []; // Also clear these on re-init
        activeClickRipples = [];   // And these
        nodesCreatedDuringGenesis = 0;

        for (let i = 0; i < PARAMS.genesisSeedNodes; i++) {
            if (nodesCreatedDuringGenesis < PARAMS.genesisMaxNodes) {
                let newNodeX, newNodeY, validPosition;
                let attempts = 0;
                const MAX_SEED_ATTEMPTS = 20; // Max attempts to find a valid position for a seed node

                do {
                    validPosition = true;
                    // Spread seed nodes a bit more widely initially
                    newNodeX = width / 2 + getRandomInRange(-width * 0.2, width * 0.2); 
                    newNodeY = height / 2 + getRandomInRange(-height * 0.2, height * 0.2);

                    // Ensure within bounds
                    newNodeX = Math.max(PARAMS.minNodeDistance, Math.min(width - PARAMS.minNodeDistance, newNodeX));
                    newNodeY = Math.max(PARAMS.minNodeDistance, Math.min(height - PARAMS.minNodeDistance, newNodeY));

                    for (const existingNode of nodes) {
                        if (Math.hypot(existingNode.x - newNodeX, existingNode.y - newNodeY) < PARAMS.minNodeDistance) {
                            validPosition = false;
                            break;
                        }
                    }
                    attempts++;
                } while (!validPosition && attempts < MAX_SEED_ATTEMPTS);

                if (validPosition) {
                    nodes.push(new Node(newNodeX, newNodeY));
                    nodesCreatedDuringGenesis++;
                }
            }
        }
    }
    
        function runGenesisStep() {
        if (Date.now() - genesisStartTime > PARAMS.genesisDuration && nodesCreatedDuringGenesis >= PARAMS.genesisMaxNodes -5) { 
            genesisInProgress = false;
            fullyConnectNetwork(nodes, PARAMS.connectionDistance, PARAMS.connectionProbability * 1.5); 
            return;
        }

        // Grow nodes
        if (nodesCreatedDuringGenesis < PARAMS.genesisMaxNodes && 
            nodes.length > 0 && // Ensure parent nodes exist
            Math.random() < PARAMS.genesisNodeGrowthRate * nodes.length) {
            
            let parentNode;
            let newX, newY, validPosition;
            let attempts = 0;
            const MAX_BUDDING_ATTEMPTS = 10; // Max attempts to find a valid spot for a new node

            do {
                validPosition = true;
                parentNode = nodes[Math.floor(Math.random() * nodes.length)];
                const angle = Math.random() * Math.PI * 2;
                // Make bud distance slightly more variable and potentially shorter to find space
                const budDist = PARAMS.genesisNodeBudDistance * getRandomInRange(0.2, 0.8); // Adjusted range
                newX = parentNode.x + Math.cos(angle) * budDist;
                newY = parentNode.y + Math.sin(angle) * budDist;

                // Check bounds first (allow some grace for radius if needed, or strict bounds)
                if (newX <= PARAMS.minNodeDistance || newX >= width - PARAMS.minNodeDistance || 
                    newY <= PARAMS.minNodeDistance || newY >= height - PARAMS.minNodeDistance) {
                    validPosition = false;
                    attempts++;
                    continue; 
                }

                // Check min distance from other nodes
                for (const existingNode of nodes) {
                    if (Math.hypot(existingNode.x - newX, existingNode.y - newY) < PARAMS.minNodeDistance) {
                        validPosition = false;
                        break;
                    }
                }
                attempts++;
            } while (!validPosition && attempts < MAX_BUDDING_ATTEMPTS);

            if (validPosition) {
                 nodes.push(new Node(newX, newY));
                 nodesCreatedDuringGenesis++;
                 const newNode = nodes[nodes.length-1];
                 // Attempt to connect new node immediately
                 for(let node of nodes){ // Iterate through all nodes, not just parent
                    if(node.id === newNode.id) continue;
                    const dist = Math.hypot(newNode.x - node.x, newNode.y - node.y);
                    // Connect if close enough and random chance, slightly higher chance for new nodes
                    if(dist < PARAMS.connectionDistance * 0.9 && Math.random() < PARAMS.connectionProbability * 1.5){ 
                        newNode.connect(node);
                    }
                 }
            }
        }

        // Grow connections (original logic is fine, can remain)
        if (nodes.length > 1 && Math.random() < PARAMS.genesisConnectionGrowthRate * nodes.length) {
            const node1 = nodes[Math.floor(Math.random() * nodes.length)];
            const node2 = nodes[Math.floor(Math.random() * nodes.length)];
            if (node1.id !== node2.id) {
                const dist = Math.hypot(node1.x - node2.x, node1.y - node2.y);
                if (dist < PARAMS.connectionDistance && Math.random() < PARAMS.connectionProbability) {
                    node1.connect(node2);
                }
            }
        }
        
        // Spawn initial pulses slowly during genesis (as modified in point 1)
        if (nodes.length > 1 && Math.random() < PARAMS.pulseSpawnRate * 4) { 
            const startNode = nodes[Math.floor(Math.random() * nodes.length)];
            const connectedNodeIds = startNode.connections;
            if (connectedNodeIds.length > 0) {
                const endNodeId = connectedNodeIds[Math.floor(Math.random() * connectedNodeIds.length)];
                const endNode = getNodeById(endNodeId);
                if(startNode && endNode) pulses.push(new Pulse(startNode, endNode));
            }
        }
    } // End of runGenesisStep

    function fullyConnectNetwork(nodeList, distance, probability) {
        for (let i = 0; i < nodeList.length; i++) {
            for (let j = i + 1; j < nodeList.length; j++) {
                const dist = Math.hypot(nodeList[i].x - nodeList[j].x, nodeList[i].y - nodeList[j].y);
                if (dist < distance && Math.random() < probability) {
                    nodeList[i].connect(nodeList[j]);
                }
            }
        }
    }

    function drawConnections(currentCognitiveLoad) {
        let baseConnectionColor = PARAMS.baseConnectionColor;
        // No specific color shift for connections in cognitive load, they just become more numerous via pulses
        
        ctx.lineWidth = PARAMS.connectionBaseWidth;

        nodes.forEach(node => {
            const connectedNodeIds = node.connections;
            connectedNodeIds.forEach(connectedNodeId => {
                const connectedNode = getNodeById(connectedNodeId);
                if (!connectedNode || nodes.indexOf(node) >= nodes.indexOf(connectedNode)) return; // Draw once, ensure node exists

                const opacity = node.opacity * connectedNode.opacity * (baseConnectionColor.match(/rgba?\([^,]+,[^,]+,[^,]+,([^)]+)\)/)?.[1] || 1);
                
                const midX = (node.x + connectedNode.x) / 2;
                const midY = (node.y + connectedNode.y) / 2;
                const dx = connectedNode.x - node.x;
                const dy = connectedNode.y - node.y;
                const angle = Math.atan2(dy, dx);
                const dist = Math.hypot(dx, dy);
                
                // Curvature: control point perpendicular to the line
                const curvature = dist * PARAMS.connectionCurvatureMax * (Math.random() - 0.1) * 2; // Randomize direction of curve
                const cpX = midX + Math.cos(angle + Math.PI / 2) * curvature;
                const cpY = midY + Math.sin(angle + Math.PI / 2) * curvature;

                ctx.beginPath();
                ctx.moveTo(node.x, node.y);
                ctx.quadraticCurveTo(cpX, cpY, connectedNode.x, connectedNode.y);
                
                const effectiveConnectionColor = baseConnectionColor.replace(/rgba?\(([^)]+)\)/, `rgba($1, ${opacity})`);
                ctx.strokeStyle = effectiveConnectionColor;
                ctx.stroke();

                if (PARAMS.connectionGlow) {
                    ctx.save();
                    ctx.lineWidth = PARAMS.connectionBaseWidth * PARAMS.connectionGlowWidthMultiplier;
                    ctx.strokeStyle = PARAMS.connectionGlowColor.replace(/rgba?\(([^)]+)\)/, `rgba($1, ${opacity * 0.5})`); // Glow is more transparent
                    ctx.shadowBlur = 5; // Optional: add a canvas shadow for more glow
                    ctx.shadowColor = PARAMS.connectionGlowColor.replace(/rgba?\(([^)]+)\)/, `rgba($1, ${opacity * 0.2})`);
                    ctx.stroke(); // Redraw wider line for glow
                    ctx.restore();
                }
            });
        });
    }

    function handleMouseEffect() {
        nodes.forEach(node => {
            let brightnessTarget = 1;
            if (mouse.x !== undefined && mouse.y !== undefined) {
                const distToMouse = Math.hypot(node.x - mouse.x, node.y - mouse.y);
                if (distToMouse < PARAMS.mouseEffectRadius) {
                    brightnessTarget = PARAMS.mouseNodeBrightnessFactor * (1 - distToMouse / PARAMS.mouseEffectRadius);
                    brightnessTarget = Math.max(1, brightnessTarget);
                }
            }
            // Smooth transition for brightness
            node.brightness += (brightnessTarget - node.brightness) * 0.1;
        });
    }
    
    function handleFocusMode() {
        const focusYStart = focusedProjectCard ? focusedProjectCard.offsetTop - window.scrollY - 50 : -1; // 50px buffer
        const focusYEnd = focusedProjectCard ? focusedProjectCard.offsetTop - window.scrollY + focusedProjectCard.offsetHeight + 50 : -1;

        nodes.forEach(node => {
            let targetOpacity = 1;
            if (focusedProjectCard) {
                const nodeInFocusZone = node.y > focusYStart && node.y < focusYEnd;
                // Simple horizontal check too, e.g. node within card's horizontal span on screen
                // For now, primarily Y-axis based focus for background elements.
                targetOpacity = nodeInFocusZone ? 1 : PARAMS.focusModeDimOpacity;
            }
            node.opacity += (targetOpacity - node.opacity) * 0.1; // Smooth opacity change
        });

        pulses.forEach(pulse => {
            let targetOpacity = 1;
            if(focusedProjectCard){
                 const pulseInFocusZone = pulse.y > focusYStart && pulse.y < focusYEnd;
                 targetOpacity = pulseInFocusZone ? 1 : PARAMS.focusModeDimOpacity;
            }
            pulse.opacity += (targetOpacity - pulse.opacity) * 0.1;
        });
    }

    function handleCognitiveLoad() {
        const now = Date.now();
        userInteractionTimestamps = userInteractionTimestamps.filter(t => now - t < PARAMS.cognitiveLoadTimeWindow);

        if (userInteractionTimestamps.length >= PARAMS.cognitiveLoadThreshold) {
            cognitiveLoadLevel = Math.min(PARAMS.cognitiveLoadMaxLevel, cognitiveLoadLevel + 0.4); // Rapid increase
            lastInteractionTime = now; // Keep track of last high activity
        } else {
            // Decay if not recently highly active
             if (now - lastInteractionTime > PARAMS.cognitiveLoadTimeWindow * 2) { // Add a buffer before decaying
                cognitiveLoadLevel = Math.max(0, cognitiveLoadLevel - PARAMS.cognitiveLoadDecayRate);
             }
        }
    }

    function drawSynapticSparks() {
        const now = Date.now();
        activeSynapticSparks = activeSynapticSparks.filter(spark => {
            const elapsed = now - spark.startTime;
            if (elapsed > PARAMS.synapticSparkDuration) return false;

            const progress = elapsed / PARAMS.synapticSparkDuration;
            const currentRadius = spark.radius * Math.sin(progress * Math.PI); // Grows and shrinks
            const opacity = Math.sin(progress * Math.PI) * 0.9; // Fades in and out

            ctx.beginPath();
            ctx.arc(spark.node.x, spark.node.y, currentRadius, 0, Math.PI * 2);
            ctx.fillStyle = PARAMS.synapticSparkColor.replace(/rgba?\(([^)]+)\)/, `rgba($1, ${opacity})`);
            ctx.fill();
            return true;
        });
    }
    
    function drawClickRipples() {
        const now = Date.now();
        activeClickRipples = activeClickRipples.filter(ripple => {
            const elapsed = now - ripple.startTime;
            if (elapsed > PARAMS.mouseClickRippleDuration) return false;

            const progress = elapsed / PARAMS.mouseClickRippleDuration;
            const currentRadius = PARAMS.mouseClickRippleMaxRadius * progress;
            const opacity = (1 - progress) * (PARAMS.mouseClickRippleColor.match(/rgba?\([^,]+,[^,]+,[^,]+,([^)]+)\)/)?.[1] || 1);


            ctx.beginPath();
            ctx.arc(ripple.x, ripple.y, currentRadius, 0, Math.PI * 2);
            ctx.strokeStyle = PARAMS.mouseClickRippleColor.replace(/rgba?\(([^)]+)\)/, `rgba($1, ${opacity})`);
            ctx.lineWidth = 2 * (1-progress); // Line thins out
            ctx.stroke();
            return true;
        });
    }


    function animate() {
        requestAnimationFrame(animate);
        ctx.clearRect(0, 0, width, height);
        frameCount++;

        if (genesisInProgress) {
            runGenesisStep();
        }
        
        handleCognitiveLoad(); // Update cognitive load level
        handleMouseEffect();
        handleFocusMode();

        drawConnections(cognitiveLoadLevel);

        nodes.forEach(node => {
            node.draw(cognitiveLoadLevel);
            // Spawn new pulses
            if (!genesisInProgress && Math.random() < PARAMS.pulseSpawnRate * (1 + cognitiveLoadLevel * PARAMS.cognitiveLoadPulseMultiplier) && node.connections.length > 0) {
                const connectedNodeIds = node.connections;
                const endNodeId = connectedNodeIds[Math.floor(Math.random() * connectedNodeIds.length)];
                const endNode = getNodeById(endNodeId);
                 if(node && endNode) pulses.push(new Pulse(node, endNode));
            }
        });

        pulses = pulses.filter(pulse => {
            const remove = pulse.update(cognitiveLoadLevel);
            if (!remove) pulse.draw(cognitiveLoadLevel);
            return !remove;
        });
        
        drawSynapticSparks();
        drawClickRipples();
    }

    // Event Listeners
    projectCards.forEach(card => {
        card.addEventListener('click', () => {
            userInteractionTimestamps.push(Date.now()); // Track for cognitive load

            if (focusedProjectCard === card) { // Click again to unfocus
                focusedProjectCard.classList.remove('focused');
                projectCards.forEach(c => c.classList.remove('dimmed'));
                focusedProjectCard = null;
            } else {
                if (focusedProjectCard) {
                    focusedProjectCard.classList.remove('focused');
                }
                focusedProjectCard = card;
                focusedProjectCard.classList.add('focused');
                projectCards.forEach(c => {
                    if (c !== focusedProjectCard) {
                        c.classList.add('dimmed');
                    } else {
                        c.classList.remove('dimmed');
                    }
                });
            }
        });
    });
    
    canvas.addEventListener('click', (event) => {
        if (PARAMS.mouseClickRipple) {
            activeClickRipples.push({ x: event.clientX, y: event.clientY, startTime: Date.now() });
        }
        if (PARAMS.mousePulseSpawnOnClick && !genesisInProgress && nodes.length > 0) {
            const numPulsesToSpawn = 3; // Spawn a few pulses
            for(let i=0; i<numPulsesToSpawn; i++){
                const randomNode = nodes[Math.floor(Math.random() * nodes.length)];
                if(randomNode.connections.length > 0){
                    const endNodeId = randomNode.connections[Math.floor(Math.random() * randomNode.connections.length)];
                    const endNode = getNodeById(endNodeId);
                    if(randomNode && endNode) pulses.push(new Pulse(randomNode, endNode));
                }
            }
        }
        userInteractionTimestamps.push(Date.now()); // Track for cognitive load
    });


    // Update current year in footer
    const currentYearSpan = document.getElementById('currentYear');
    if (currentYearSpan) currentYearSpan.textContent = new Date().getFullYear();

    // Smooth scroll for navigation links
    document.querySelectorAll('nav a[href^="#"]').forEach(anchor => {
        anchor.addEventListener('click', function (e) {
            e.preventDefault();
            const targetElement = document.querySelector(this.getAttribute('href'));
            if (targetElement) {
                targetElement.scrollIntoView({
                    behavior: 'smooth'
                });
            }
        });
    });

    window.addEventListener('mousemove', (event) => {
        mouse.x = event.clientX;
        mouse.y = event.clientY;
    });
    
    window.addEventListener('mouseout', () => {
        mouse.x = undefined;
        mouse.y = undefined;
        nodes.forEach(node => node.brightness = 1); // Reset brightness
    });

    window.addEventListener('resize', resizeCanvas);
    
    // Initial Setup
    resizeCanvas(); // This will call initGenesisNetwork
    animate();
});