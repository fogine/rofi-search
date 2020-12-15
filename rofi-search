#!/bin/sh
':' //; exec "$(command -v nodejs || command -v node)" "$0" "$@"


'use strict';

const child_process = require('child_process');
const path          = require('path');
const EventEmitter  = require('events');
const fs            = require('fs');
const https         = require('https')

const APP_NAME = 'rofi-search';
const $HOME = process.env.HOME;
const $XDG_CACHE_HOME = process.env.XDG_CACHE_HOME;
const TITLE_COLOR = process.env.TITLE_COLOR || '#6CB6DA';
const ROFI_SEARCH_TIMEOUT = parseInt(process.env.ROFI_SEARCH_TIMEOUT) || 500;
const ROFI_SEARCH_CMD = process.env.ROFI_SEARCH_CMD;

const CACHE_DIR_PATH = ($XDG_CACHE_HOME || ($HOME + "/.cache")) + '/' + APP_NAME;
const CACHE_FILE_PATH = `${CACHE_DIR_PATH}/last_search.json`;

const SearchMethods = Object.freeze({
    CSE: 'cse',
    GOOGLER: 'googler',
    DDGR: 'ddgr'
});

/**
 */
function main() {
    if (!process.env.ROFI_SEARCH) {
        return runRofiSearchWithDeafultConfig();
    }

    const rofi = new Rofi({
        titleColor: TITLE_COLOR
    });

    const search = new Search({
        apiKey: process.env.GOOGLE_API_KEY,
        //search engine id
        cx: process.env.GOOGLE_SEARCH_ID,
        rofi: rofi
    });

    search.on('error', function(err) {
        if (isDebugModeEnabled()) { //verbose logging
            console.error(err);
        } else {
            console.error(err.message);
        }
        process.exit(1);
    });

    search.on('search-result', function(data) {
        rofi.setData(data);
        rofi.updateListView();
        rofi.updateMessageView();
    });

    //load last search query results
    rofi.setData(search.loadCache())
    rofi.updateListView();
    rofi.updateMessageView();
    //initialize rofi if there is no cached last usage data
    if (rofi.data === null) {
        rofi.write({});
    }

    process.stdin.setEncoding('utf8');
    process.stdin.on('readable', function() {
        let data;

        while (data = this.read()) {

            const messages = data.split('\n');
            messages.forEach(function(message) {
                if (message) {
                    message = JSON.parse(message);
                    if (isDebugModeEnabled()) {
                        console.error(`emmiting event name ${message.name}`);
                        console.error(`emmiting event value ${message.value}`);
                    }
                    this.emit(message.name, message.value);
                }
            }, this);
        }
    });

    process.stdin.on('input change', function(value) {
        search.query(value);
    });

    process.stdin.on('select entry', function(value) {
        if (search.q !== search.last_q) {
            /*
             * if search query has changed, wait until search results are updated
             * and then select first entry
             */
            return rofi.queueSelectFirstSearchResult();
        }
        const searchResult = rofi.getMatchingRecord(value);
        xdgOpen(searchResult.link);
        process.exit(0);
    });

    process.stdin.on('execute custom input', function(value) {
        if (!value || !rofi.data) {
            /*
             * Makes sure that the first search result is opened when enter
             * is pressed after search query.
             * This handles the case where enter is pressed before search
             * results are available which would normally result in
             * opening google page with the search query
             */
            return rofi.queueSelectFirstSearchResult();
        }
        //ctrl-enter executes search query on google.com
        xdgOpen(getGoogleSearchUrl(value));
        process.exit(0);
    });

    process.stdin.on('active entry', function(value) {
        this._tempBuffer = value;
    });

    process.stdin.on('custom key', function(value) {
        const selectedEntry = rofi.getMatchingRecord(this._tempBuffer);
        switch (value) {
            //copy to clipboard
            case '1': //-kb-custom-1
                const xclip = child_process.spawn('xclip', ['-l', 1]);
                xclip.on('close', exit);

                xclip.stdin.end(selectedEntry.link);
                break;
            default:
        }
    });
}


/*
 *
 *===========================ROFI IPC=========================
 */

/**
 * @param {Object} config
 * @param {String} config.titleColor
 */
function Rofi(config) {
    this.config = config || {};
    this.data = null; // object with search api results
    this.message = {
        prompt: 'search',
        "input action": "send"
    };
    this.queue = [];
    this.timeoutID = null;
    process.stdout.setDefaultEncoding('utf8');
}

Rofi.prototype = Object.create(EventEmitter.prototype);
Rofi.prototype.constructor = Rofi;

/**
 * @param {Object} data
 * @return {undefined}
 */
Rofi.prototype.setData = function(data) {
    this.data = data;
};


/**
 * @return {Object|null}
 */
Rofi.prototype.getFirstSearchRecord = function() {
    if (!this.data || !Array.isArray(this.data.items)) {
        return null;
    }
    return this.data.items[0];
};


/**
 * getMatchingRecord
 * @param {String} selectedText - rofi formatted entry
 * @return {Object|null}
 */
Rofi.prototype.getMatchingRecord = function(selectedText) {
    if (!this.data || !Array.isArray(this.data.items)) {
        return null;
    }
    for (var i = 0, len = this.message.lines.length; i < len; i++) {
        if (this.message.lines[i].text == selectedText) {
            break;
        }
    }

    return this.data.items[i];
};


/**
 * @return {undefined}
 */
Rofi.prototype.queueSelectFirstSearchResult = function() {
    return this.once('after-update', function() {
        const searchResult = this.getFirstSearchRecord();
        xdgOpen(searchResult.link);
        process.exit(0);
    });
};

/**
 * write
 * @param {Object} data
 */
Rofi.prototype.write = function(data) {
    this.queue.push(data);

    if (this.timeoutID) {
        clearTimeout(this.timeoutID);
    }

    this.timeoutID = setTimeout(() => {
        this.queue.unshift(this.message);
        Object.assign.apply(null, this.queue);

        //console.error(JSON.stringify(this.message));
        process.stdout.write(JSON.stringify(this.message));
        process.stdout.write('\n');
        this.emit('after-update');
    }, 0);
};

/**
 * @return {undefined}
 */
Rofi.prototype.updateMessageView = function updateMessageView() {
    if (this.data === null) {
        return;
    }

    const context = this.data.searchInformation;
    let message='';
    if (context.formattedTotalResults) {
        message = `<span size="x-small">About ${context.formattedTotalResults} results (${context.formattedSearchTime} seconds)</span>`;
    }

    this.write({message: message});
};

/**
 * @return {undefined}
 */
Rofi.prototype.updateListView = function updateListView() {
    if (this.data === null || !Array.isArray(this.data.items)) {
        return;
    }

    const searchResults = [];
    this.data.items.forEach(function(item, index) {
        let text = '';

        let snippetSegments = item.snippet.replace(/\n/g, '').split('...').filter(trimArr).map(trimStr);
        let snippet = '';

        if (snippetSegments.length == 2) {
            snippet = snippetSegments.join('\n');
        } else if (snippetSegments.length > 2) {
            snippet = snippetSegments[0] + ' ... ' + snippetSegments[1] + '\n';
            snippet += snippetSegments.slice(2).join(' ... ').trim();
        } else if (snippetSegments.length) {
            snippet = snippetSegments[0];
        }

        if (snippet && snippet[snippet.length-1] != '.') {
            snippet += ' ...';
        }

        let link = prettyPrintUrl(escapePangoMarkup(item.link));
        let title = escapePangoMarkup(item.title);
        snippet = escapePangoMarkup(snippet);

        text += '<span size="x-small" rise="10000">' + link + '</span>\n';
        text += `<span color="${this.config.titleColor}" rise="10000">` + title + '</span>\n';
        text += '<span size="x-small">' + snippet + '</span>';
        //console.error(text);

        searchResults.push({
            text: text,
            urgent: index < 2 ? true : false,
            markup: true
        });
    }, this);

    this.write({lines: searchResults});
};

function escapePangoMarkup(unsafe) {
    return unsafe
         .replace(/&/g, "&amp;")
         .replace(/</g, "&lt;")
         .replace(/>/g, "&gt;")
         .replace(/"/g, "&quot;")
         .replace(/'/g, "&#039;");
 }

function trimStr(str) {
    return str.trim();
}

function trimArr(item) {
    return item !== null && item !== '' && item !== undefined;
}

/**
 * @param {String} url
 * @return {String}
 */
function prettyPrintUrl(url) {
    let prettyUrl = '';
    url.split('/').slice(2).forEach(function(segment, index, arr) {
        if (index == 0) {
            segment = '<b>' + segment + '</b>';
        }
        prettyUrl += segment;
        if (arr[index+1]) {
            prettyUrl += ' › ';
        }
    });
    return prettyUrl;
}


/*
 *
 *===========================GOOGLE/DDG SEARCH=========================
 */

/**
 * @param {Object} config
 * @param {String} config.apiKey
 * @param {String} config.cx
 */
function Search(config) {
    this.options = {
        key: config.apiKey,
        cx: config.cx
    };
    this.q = ''; //current query
    this.last_q = ''; //last query
    this.timeoutID = null;
}

Search.prototype = Object.create(EventEmitter.prototype);
Search.prototype.constructor = Search;

/**
 * @param {String} q
 */
Search.prototype.query = function query(q) {

    const self = this;
    this.q = q;
    const options = Object.assign({q}, this.options);

    if (this.timeoutID) {
        clearTimeout(this.timeoutID);
    }

    this.timeoutID = setTimeout(_search, ROFI_SEARCH_TIMEOUT);

    async function _search() {
        try {
            //ddgr crashes when invoced with empty string
            if (!q) {
                return;
            }
            const data = await self.getSearchResults(options);
            self.last_q = q;
            self.emit('search-result', data);
            self.saveCache(data);
        } catch(e) {
            self.emit('error', e);
        }
    }
}


/**
 * @param {Object} options
 * @return {Promise<Object>}
 */
Search.prototype.getSearchResults = async function getSearchResults(options) {
    const self = this;
    const engines = getPreferedSearchMethods();

    let promiseList = engines.reduce(function(promiseList, engine) {
        let args, p;
        switch (engine) {
            case SearchMethods.CSE:
                args = parseJsonEnv('GOOGLE_ARGS');
                if (args.includes('--count')) {
                    options.num = args[args.indexOf('--count')+1];
                } else if (args.includes('-n')) {
                    options.num = args[args.indexOf('-n')+1];
                }

                p = self.customEngineSearch(options);
                break;
            case SearchMethods.GOOGLER:
                args = parseJsonEnv('GOOGLE_ARGS').concat(['--json', `${options.q}`]);
                p = self._spawnChildProcess(SearchMethods.GOOGLER, args).then(function(data) {
                    return self._parseGooglerResults(data);
                });
                break;
            case SearchMethods.DDGR:
                args = parseJsonEnv('DDG_ARGS').concat(['-x','--json', `${options.q}`]);
                p = self._spawnChildProcess(SearchMethods.DDGR, args).then(function(data) {
                    return self._parseDDGRResults(data);
                });
                break;
        }

        if (p) {
            promiseList.push(p);
        }
        return promiseList;
    }, []);

    return Promise.all(promiseList).then(function(results) {
        //return Array.prototype.concat.apply([], results);
        return results.reduce(function(flatResults, engineResults) {
            Array.prototype.push.apply(flatResults.items, engineResults.items);
            return flatResults;
        }, {
            searchInformation: results[0].searchInformation,
            items: []
        });
    }).then(function(data) {
        return self._removeDuplicates(data);
    });
};

/**
 * removes search result duplicates
 * @param {Object} data
 * @return {Object}
 */
Search.prototype._removeDuplicates = function _removeDuplicates(data) {
    let urlBuffer = [];
    data.items = data.items.filter(function(item) {
        //remove duplicates
        if (urlBuffer.includes(item.link)) {
            return false;
        }
        urlBuffer.push(item.link);
        return true
    });
    return data;
};


/**
 * @return {Object}
 */
Search.prototype._parseGooglerResults = function _parseGooglerResults(data) {
    data = JSON.parse(data).map(function(item) {
        if (item.metadata) {
            let metadataSegments = item.metadata.split(',');
            //prepend data information
            //so its in the same format as search results from google API
            item.abstract =  metadataSegments[0] + ' ... ' + item.abstract;
        }
        return {
            link: item.url,
            snippet: item.abstract,
            title: item.title
        };
    }).filter(trimArr);

    data = {
        searchInformation: {
            formattedSearchTime: '',
            formattedTotalResults: ''
        },
        items: data
    };

    return data;
};

/**
 * @return {Object}
 */
Search.prototype._parseDDGRResults = function _parseDDGRResults(data) {
    //TODO remove once ddgr version >1.7-1 will be released
    data = data.slice(data.indexOf('[\n  {'));
    return this._parseGooglerResults(data);
};


/**
 * @param {String} executable - googler|ddgr
 * @param {Array<String>} args - to be provided to the executable
 * @return {Promise<Object>}
 */
Search.prototype._spawnChildProcess = function(executable, args) {
    return new Promise(function(resolve, reject) {
        let stdout = '';
        let stderr = '';

        const proc = child_process.spawn(executable, args);

        proc.stdout.on('data', function(chunk) {
            stdout += chunk.toString();
        });

        proc.stderr.on('data', function(chunk) {
            stderr += chunk.toString();
        })

        proc.once('close', function(status) {
            if (status !== 0) {
                return reject(stderr);
            }

            return resolve(stdout);
        });

        proc.once('error', function(err) {
            if (err.code === 'ENOENT') {
                return reject(new Error(`Can not find "${executable}" executable.`));
            }
            return reject(err);
        });
    });
};

/**
 * @param {Object} options
 * @return {Promise}
 */
Search.prototype.customEngineSearch = function(options) {
    return new Promise(function(resolve, reject) {
        let query = '';
        Object.keys(options).forEach(function(prop) {
            query += prop+'='+encodeURIComponent(options[prop])+'&';
        });
        const opt = {
            hostname: 'www.googleapis.com',
            port: 443,
            path: `/customsearch/v1?${query}`,
            method: 'GET',
            headers: {
                'x-goog-api-client': 'gdcl/3.2.2 gl-node/8.16.2 auth/5.10.1',
                Accept: 'application/json',
                //'Accept-Encoding': 'gzip',
                //'User-Agent': 'google-api-nodejs-client/3.2.2 (gzip)',
            }
        }
        let data = '';

        const req = https.request(opt, res => {

            res.on('data', chunk => {
                data += chunk;
            });

            res.on('end', () => {
                //console.error('===============');
                //console.error(`statusCode: ${res.statusCode}`)
                //console.error(data)

                //wierd undefined behavior:
                //when executing this via i3wm shortcut mapping,
                //statusCode is 65535 instead of 200
                if ((res.statusCode >= 200 && res.statusCode < 300)
                    || res.statusCode === 65535
                ) {
                    resolve(JSON.parse(data));
                } else {
                    reject(data);
                }
            });
        });

        req.on('error', error => {
            reject(error);
        });

        req.end();
    });
};

/**
 * @return {undefined}
 */
Search.prototype.saveCache = function saveCache(data) {
    if (!fs.existsSync(CACHE_DIR_PATH)) {
        fs.mkdirSync(CACHE_DIR_PATH);
    }
    fs.writeFileSync(CACHE_FILE_PATH, JSON.stringify(data), 'utf-8');
}

/**
 * @return {Object|null}
 */
Search.prototype.loadCache = function loadCache() {
    if (fs.existsSync(CACHE_FILE_PATH)) {
        return JSON.parse(fs.readFileSync(CACHE_FILE_PATH));
    }

    return null;
}

/*
 *
 *===========================UTILS=========================
 */

/**
 * @return {boolean}
 */
function isDebugModeEnabled() {
    return !!process.env.ROFI_SEARCH_DEBUG;
}

/**
 * @return {Array}
 */
function parseJsonEnv(envVarName) {

    let args = process.env[envVarName] || '[]';
    try {
        args = JSON.parse(args);
        if (!Array.isArray(args)) {
            throw new Error(`${envVarName} env variable is not valid json array`);
        }
        return args;
    } catch(e) {
        console.error(`Failed to parse ${envVarName} env variable.`);
        console.error(e);
        process.exit(1);
    }
}

/**
 * @return {Array<String>}
 */
function getPreferedSearchMethods() {
    let methods = (process.env.ROFI_SEARCH || '').split(',');
    let supportedMethods = [
        SearchMethods.CSE,
        SearchMethods.GOOGLER,
        SearchMethods.DDGR
    ];

    methods = methods.filter(function(searchMethod) {
        return supportedMethods.includes(searchMethod);
    });

    if (methods.length) {
        return methods;
    } else {
        //return search method fallback
        if (process.env.GOOGLE_API_KEY && process.env.GOOGLE_SEARCH_ID) {
            return [SearchMethods.CSE];
        } else {
            return [SearchMethods.GOOGLER];
        }
    }
}

/**
 * @param {String} url
 * @return stdout
 */
function xdgOpen(url) {
    if (ROFI_SEARCH_CMD) {
        child_process.exec(ROFI_SEARCH_CMD, {
            env: Object.assign({URL: url}, process.env)
        }, exit);
    } else {
        const xdgOpen = child_process.spawn('xdg-open', [url]);
        xdgOpen.on('close', exit);
    }
}

function exit() {
    process.exit(0);
}

/**
 * @param {String} q - search query
 * @return {String}
 */
function getGoogleSearchUrl(q) {
    return `https://google.com/search?q=${encodeURIComponent(q)}`;
}


/**
 * @return {undefined}
 */
function runRofiSearchWithDeafultConfig() {

    const theme = getDefaultThemeStr();
    const env = {
        GOOGLE_ARGS:'["--count", 10]',
        DDG_ARGS:'["-n", 10]',
        ROFI_SEARCH: getDefaultSearchMethod()
    };

    const args = ['-modi', 'blocks', '-show', 'blocks', '-lines', 4, '-eh', 4,
        '-kb-custom-1', 'Control+y', '-theme-str', theme,
        '-blocks-prompt', 'search', '-blocks-wrap', __filename
    ];
    child_process.spawn('rofi', args, {
        env: Object.assign(env, process.env),
        detached: true,
        stdio: ['inherit', 'inherit', 'inherit']
    });

}

/**
 * @return {String}
 */
function getDefaultSearchMethod() {
    try {
        child_process.execSync(`which ${SearchMethods.GOOGLER}`, {stdio: [null, null, null]})
        return SearchMethods.GOOGLER;
    } catch(e) {
        try {
            child_process.execSync(`which ${SearchMethods.DDGR}`, {stdio: [null, null, null]})
            return SearchMethods.DDGR;
        } catch(e) {
            console.error(`To make rofi-search work with default configuration, please install either ddgr - https://github.com/jarun/ddgr or googler - https://github.com/jarun/googler`);
            process.exit(1);
        }
    };
}

/**
 * @return {String}
 */
function getDefaultThemeStr() {
    return '*{darker:#303030;lighter:rgba(38,38,38,100%);highlight:#404040;separatorcolor:var(darker);scrollbarcolor:var(highlight);border-color:var(darker);normal-foreground:var(foreground);background:var(lighter);active-background:var(darker);normal-background:var(darker);urgent-background:var(darker);selected-normal-background:var(highlight);alternate-normal-background:var(darker);alternate-active-background:var(darker);alternate-normal-foreground:var(foreground);alternate-active-foreground:var(active-foreground);alternate-urgent-foreground:var(urgent-foreground);/**/red:rgba(220,50,47,100%);selected-active-foreground:rgba(56,168,242,100%);lightfg:rgba(88,104,117,100%);urgent-foreground:rgba(243,132,61,100%);alternate-urgent-background:rgba(38,38,38,100%);lightbg:rgba(238,232,213,100%);spacing:2;background-color:rgba(0,0,0,0%);active-foreground:rgba(0,85,119,100%);blue:rgba(38,139,210,100%);selected-active-background:rgba(0,85,119,100%);selected-normal-foreground:rgba(255,255,255,100%);foreground:rgba(255,255,255,100%);selected-urgent-background:rgba(0,85,119,100%);selected-urgent-foreground:rgba(243,132,61,100%);}window{padding:5;background-color:var(background);border:1;width:100%;height:50%;anchor:south;location:south;}mainbox{padding:0;border:0;}message{padding:10px1px;border-color:var(separatorcolor);border:2pxsolid0px0px0px;}textbox{text-color:var(foreground);}listview{padding:2px0px0px;scrollbar:true;border-color:var(separatorcolor);spacing:5px;fixed-height:0;border:0px0px0px0px;}element{padding:10px;border:0;highlight:boldunderline#96CAE4;}elementnormal.normal{background-color:var(normal-background);text-color:var(normal-foreground);}elementnormal.urgent{background-color:var(urgent-background);text-color:var(urgent-foreground);}elementnormal.active{background-color:var(active-background);text-color:var(active-foreground);}elementselected.normal{background-color:var(selected-normal-background);text-color:var(selected-normal-foreground);}elementselected.urgent{background-color:var(selected-urgent-background);text-color:var(selected-urgent-foreground);}elementselected.active{background-color:var(selected-active-background);text-color:var(selected-active-foreground);}elementalternate.normal{background-color:var(alternate-normal-background);text-color:var(alternate-normal-foreground);}elementalternate.urgent{background-color:var(alternate-urgent-background);text-color:var(alternate-urgent-foreground);}elementalternate.active{background-color:var(alternate-active-background);text-color:var(alternate-active-foreground);}scrollbar{width:4px;padding:0;handle-width:8px;border:0;handle-color:var(scrollbarcolor);}mode-switcher{border-color:var(separatorcolor);border:2pxdash0px0px;}button{spacing:0;text-color:var(normal-foreground);}buttonselected{background-color:var(selected-normal-background);text-color:var(selected-normal-foreground);}inputbar{padding:1px;spacing:0px;text-color:var(normal-foreground);children:[prompt,textbox-prompt-colon,entry,overlay,case-indicator];}case-indicator{spacing:0;text-color:var(normal-foreground);}entry{spacing:0;text-color:var(normal-foreground);}prompt{spacing:0;text-color:var(normal-foreground);}textbox-prompt-colon{margin:0px0.3000em0.0000em0.0000em;expand:false;str:":";text-color:inherit;}error-message{background-color:rgba(0,0,0,0%);text-color:var(normal-foreground);}';
}

main();
