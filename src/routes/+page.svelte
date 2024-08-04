<script lang="ts">
    interface Node {
        x: number,
        y: number,

        coast_of: string | null,

        is_sea: boolean,
        label: string,
    }

    interface Line {
        node1: string,
        node2: string
    }

    let image: string | null = null;
    let nodes: Record<string, Node> = {};
    let army_lines: Record<string, Line> = {};
    let fleet_lines: Record<string, Line> = {}

    function load() {
        image       = JSON.parse(localStorage.getItem("image"));
        nodes       = JSON.parse(localStorage.getItem("nodes"));
        army_lines  = JSON.parse(localStorage.getItem("army_lines"));
        fleet_lines = JSON.parse(localStorage.getItem("fleet_lines"));

        save = (key: string, obj: any) => {
            if (typeof localStorage == "object") {
                localStorage.setItem(key, JSON.stringify(obj));
            }
        };
    }

    let save = (key: string, obj: any) => {};

    $: save("image", image);
    $: save("nodes", nodes);
    $: save("army_lines", army_lines);
    $: save("fleet_lines", fleet_lines);
    
    let mode: "fleet" | "army" = "army";
    let active_lines: Record<string, Line> = army_lines;
    $: {
        if (mode == "fleet") {
            active_lines = fleet_lines;
        } else {
            active_lines = army_lines;
        }
    }

    let activeNode: string | null = null;
    let selectedNode: string | null = null;

    let coastMode = false;
    let tempLine: { x : number, y :number} | null = null;

    let img: HTMLImageElement;
    let input: HTMLInputElement
    let container: HTMLElement; 

    function getLine(lines: Record<string, Line>, node1: string, node2: string) : string | null {
        for (let line_id in lines) {
            let line = lines[line_id];
            if ((line.node1 == node1 && line.node2 == node2) || (line.node2 == node1 && line.node1 == node2)) {
                return line_id;
            }
        }
        return null;
    }

    function removeLines(lines: Record<string, Line>, node: string) : Record<string, Line> {
        for (let line_id in lines) {
            let line = lines[line_id];
            if (line.node1 == node || line.node2 == node) {
                delete lines[line_id];
            }
        }

        return lines;
    }

    function onmousemove(evt: MouseEvent) {
        if (activeNode != null) {
            nodes[activeNode].x = evt.pageX;
            nodes[activeNode].y = evt.pageY;
        }
        if (tempLine != null) {
            tempLine.x = evt.pageX;
            tempLine.y = evt.pageY;
        }
    }

    function onload() {
        container.style.width = img.width + "px";
        container.style.height = img.height + "px";
    }
;
    async function onchange() {
        let file = input.files[0];
        if (file) {
            let text = await file.text();
            image = "data:image/svg+xml;base64," + btoa(text);
        }
    }

    function genId() : string {
        return Math.random().toString(36).slice(2);
    }

    function onmousedown(evt: MouseEvent) {
        let id = genId();
        nodes[id] = {
            x : evt.pageX,
            y : evt.pageY,

            coast_of: null,
            is_sea: false,
            label: "node-" + id.slice(0, 4)
        };

        selectedNode = id;
        coastMode = false;
    }

    function nodeMouseDown(evt: MouseEvent, node_id: string) {
        if (evt.button == 2) {
            selectedNode = node_id;
            coastMode = false;

            tempLine = {
                x : evt.pageX,
                y: evt.pageY
            };

            if (keydown.c) {
                coastMode = true;
            }

            evt.preventDefault();
            return false;
        } else if (evt.button == 0) {
            activeNode = selectedNode = node_id;
            coastMode = false;
        }
    }

    function nodeMouseUp(evt: MouseEvent, node_id: string) {
        if (tempLine && selectedNode && selectedNode != node_id) {
            if (getLine(active_lines, selectedNode, node_id) != null) {
                // line already exists
                return;
            }
            if (nodes[node_id].coast_of == selectedNode || nodes[selectedNode].coast_of == node_id) {
                return;
            }

            if (coastMode) {
                let from_node = selectedNode;
                setTimeout(() => {
                    nodes[from_node].coast_of = node_id;
                }, 100);
                coastMode = false;
            } else {
                let line_id = genId();
                active_lines[line_id] = {
                    node1: selectedNode,
                    node2: node_id
                };

                army_lines = army_lines;
                fleet_lines = fleet_lines;
            }
        }
    }

    function onmouseup() {
        activeNode = tempLine = null;

        if (coastMode && selectedNode) {
            let node_id = selectedNode;
            nodes[node_id].coast_of = null;
        }
    }

    let keydown: Record<string, boolean> = {};
    function onkeyup(e: KeyboardEvent) {
        keydown[e.key] = false;
    }
    function onkeydown(e: KeyboardEvent) {
        if ((document.activeElement instanceof HTMLTextAreaElement) || (document.activeElement instanceof HTMLInputElement)) {
            return;
        }

        keydown[e.key] = true;

        if (e.key == "Backspace" || e.key == "Delete") {
            if (selectedNode) {
                let nodesToDelete = [selectedNode];
                for (let node_id in nodes) {
                    if (nodes[node_id].coast_of == selectedNode) {
                        nodesToDelete.push(node_id);
                    }
                }

                selectedNode = null;

                for (let nodeToDelete of nodesToDelete) {
                    delete nodes[nodeToDelete];

                    army_lines = removeLines(army_lines, nodeToDelete);
                    fleet_lines = removeLines(fleet_lines, nodeToDelete);
                }

                nodes = nodes;
            }
        }
    }

    function exprt() {
        let id = genId().slice(0, 5);

        function noderepr(node_id: string) : Array<[string, string]> {
            let node = nodes[node_id];
            if (node.coast_of == null) {
                return [[node.label, ""]];
            } else {
                return [[nodes[node.coast_of].label, ""], [nodes[node.coast_of].label, node.label]];
            }
        }

        {
            let labels = Object.values(nodes).filter(n => n.coast_of == null).map(n => n.label);
            let output = new Set();
            for (let label of labels) {
                if (output.has(label)) {
                    alert("Can't export: duplicate label: " + label);
                    return;
                }
                output.add(label);
            }
        }

        let json = {
            provinces: {},
            fleet_adj: [],
            army_adj: []
        };

        for (let node_id in nodes) {
            let node = nodes[node_id];
            if (node.coast_of != null) {
                continue;
            }

            json.provinces[node.label] = { 
                is_sea: node.is_sea,
                coasts: Object.keys(nodes).filter(cid => nodes[cid].coast_of == node_id).map(cid => nodes[cid].label),
            };
        }

        for (let line_id in army_lines) {
            let line = army_lines[line_id];
            json.army_adj.push([nodes[line.node1].label, nodes[line.node2].label]);
            json.army_adj.push([nodes[line.node2].label, nodes[line.node1].label]);
        }

        for (let line_id in fleet_lines) {
            let line = fleet_lines[line_id];

            let n1 = noderepr(line.node1);
            let n2 = noderepr(line.node2);

            for (let tuple1 of n1) {
                for (let tuple2 of n2) {
                    json.fleet_adj.push([tuple1, tuple2]);
                    json.fleet_adj.push([tuple2, tuple1]);
                }
            }
        }

        let a = document.createElement("a");
        a.className = "hide";
        document.body.appendChild(a);

        a.href = "data:application/json;base64," + btoa(JSON.stringify(json));
        a.download = "adj-" + id + ".json";
        a.click();

        let positions: Record<string, {x : number, y: number}> = {};
        for (let node of Object.values(nodes)) {
            if (node.coast_of) {
                positions[nodes[node.coast_of].label + "-" + node.label] = { x: node.x, y: node.y };
            } else {
                positions[node.label] = { x: node.x, y: node.y };
            }
        }

        setTimeout(() => {
            a.href = "data:application/json;base64," + btoa(JSON.stringify(positions));
            a.download = "pos-" + id + ".json";
            a.click();
        }, 500);
    }
</script>

<svelte:head>
    <title>Diplomacy Adjacency Graph Editor</title>
</svelte:head>


<style>
    :global(body) {
        padding: 0; margin: 0;
        font-family: sans-serif;
    }

    .node {
        width: 16px;
        height: 16px;
        border-radius: 8px;
        background: #555;
        cursor: pointer;

        z-index: 3;

        position: absolute; }
    .node.sea {
        background: hsl(240, 50%, 50%); }
    .node.coast {
        background: hsl(200, 50%, 45%); }
    .node.selected {
        background: hsl(330, 50%, 45%); }
    img {
        user-select: none;
-moz-user-select: none;
-khtml-user-select: none;
-webkit-user-select: none;
-o-user-select: none;
    }

    .panel, #export-button {
        z-index: 3; }
    .panel {
        position: fixed;
        padding: 16px;
        background: hsl(200, 10%, 10%);
        color: #fff;
    }
    .panel h2 {
        font-size: 20px;
        margin: 0;
        margin-bottom: 8px;
        text-align: center;
    }
    .flex {
        align-items: center;
        display: flex;
        flex-direction: row; }
    .flex > :first-child { flex-grow: 1; }

    .entry {
        color: #fff;
        padding: 4px 8px;
        background: transparent;
        border: 2px solid rgba(128,128,128,0.5); }
    .entry::placeholder {
        color: rgba(255,255,255,0.5); }
    .entry:focus {
        border-color: hsl(330, 50%, 45%);
        outline: 0;
    }

    svg {
        position: absolute;
        top: 0;
        left: 0;
        width: 100%;
        height: 100%;
        pointer-events: none;
    }

    line {
        stroke: hsl(0, 0%, 40%, 0.5);
        stroke-width: 4px; }
    line.coast {
        stroke: hsla(200, 50%, 40%, 0.5); }

    #mode-panel button {
        border: 0; background: transparent;
        color: #fff;
        padding: 4px 8px;
    }
    #mode-panel button.selected {
        background:hsl(330, 50%, 45%);
    }
    
    .tag {
        display: inline-block;
        padding: 4px 8px;
        background: rgba(255,255,255,0.25);
        margin-left: 4px;
        font-size: 13px;
    }

    .delete {
        display: inline-block;
        color: #fff;
        text-decoration: none;
        cursor: pointer;
    }
    .delete::before {
        content: '×';
    }

    .hide {
        display: none;
    }

    .button {
        color: #fff;
        border: 0;
        background: hsl(330, 50%, 45%);
        padding: 4px 8px; }
    .button:active {
        background: hsl(330, 50%, 35%); }
    #export-button {
        position: fixed;
        bottom: 16px;
        right: 16px;
    }
</style>

<svelte:window on:load={load} on:keydown={onkeydown} on:keyup={onkeyup} on:mousemove={onmousemove} on:mouseup={onmouseup} />

<main bind:this={container} class:hide={image == null}>
    <img on:mousedown={onmousedown} src={image} id="img" bind:this={img} on:load={onload} />
    {#each Object.keys(nodes) as node_id}
        {@const node = nodes[node_id]}
        <div class="node" class:selected={node_id == selectedNode} class:sea={node.is_sea}
            class:coast={node.coast_of != null}
            on:mouseup={(e) => nodeMouseUp(e, node_id)}
            on:mousedown={(e) => nodeMouseDown(e, node_id)}
            on:contextmenu={(e) => { e.preventDefault(); return false}}
            id={"node-" + node_id} style={"top:" + (node.y-8) + "px;left:" + (node.x-8) + "px"}></div>
    {/each}

    <div class="panel" id="mode-panel" style="top:16px;left:16px;right:auto;display:flex;flex-direction:row;padding:0">
        <button class:selected={mode == "army"} on:click={() => mode = "army"}>Army</button>
        <button class:selected={mode == "fleet"} on:click={() => mode = "fleet"}>Fleet</button>
    </div>

    {#if selectedNode != null}
        <div class="panel" style="bottom:16px;left:16px;width:384px">
            <h2>Node Information</h2>
            <div class="flex">
                <label for="label">{nodes[selectedNode].coast_of ? "Coast name" : "Abbreviation"}</label> <input class="entry" id="label" bind:value={nodes[selectedNode].label} />
            </div>
            {#if nodes[selectedNode].coast_of != null}
            <div class="flex">
                <label>Coast of</label> <span class="tag">{nodes[nodes[selectedNode].coast_of].label} <a class="delete" on:click={() => nodes[selectedNode].coast_of = null}></a></span>
            </div>
            {/if}
            <div class="flex">
                <label for="sea">Sea Province</label> <input id="sea" bind:checked={nodes[selectedNode].is_sea} type="checkbox" />
            </div>
            <div class="flex">
                <label for="adj">Land adjacencies</label>
                {#each Object.entries(army_lines)
                    .filter(([s, l]) => l.node1 == selectedNode || l.node2 == selectedNode)
                    .map(([s, l]) => (l.node1 == selectedNode ? [s, l] : [s, { node1 : l.node2, node2: l.node1 }])) as [line_id, l]}
                    <span class="tag">{nodes[l.node2].label} <a class="delete" on:click={() => {delete army_lines[line_id]; army_lines = army_lines}}></a></span>
                {/each}
            </div>
            <div class="flex">
                <label for="adj">Sea adjacencies</label>
                {#each Object.entries(fleet_lines)
                    .filter(([s, l]) => l.node1 == selectedNode || l.node2 == selectedNode)
                    .map(([s, l]) => (l.node1 == selectedNode ? [s, l] : [s, { node1 : l.node2, node2: l.node1 }])) as [line_id, l]}
                    <span class="tag">{nodes[l.node2].label} <a class="delete" on:click={() => {delete fleet_lines[line_id]; fleet_lines = fleet_lines }}></a></span>
                {/each}
            </div>
        </div>
    {/if}

    <svg>
        {#if tempLine && selectedNode}
            <line class:coast={coastMode} x1={nodes[selectedNode].x} y1={nodes[selectedNode].y} x2={tempLine.x} y2={tempLine.y} />
        {/if}
        {#each Object.keys(active_lines) as line_id} 
            {@const line = active_lines[line_id]}
            {@const node1 = nodes[line.node1]}
            {@const node2 = nodes[line.node2]}
            <line x1={node1.x} y1={node1.y} x2={node2.x} y2={node2.y} />
        {/each}

        {#each Object.keys(nodes) as node_id} 
            {#if nodes[node_id].coast_of}
                {@const node2 = nodes[nodes[node_id].coast_of]}
                <line class:coast={true} x1={nodes[node_id].x} y1={nodes[node_id].y} x2={node2.x} y2={node2.y} />
            {/if}
        {/each}
    </svg>

    <button class="button" on:click={exprt} id="export-button">Export</button>
</main>
<input type="file" id="input" on:change={onchange} bind:this={input} class:hide={image != null} />
