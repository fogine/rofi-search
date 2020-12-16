# rofi-search
Interactive web search via [rofi](https://github.com/DaveDavenport/rofi/).  
Multiple search engines supported.

[![Preview](https://github.com/fogine/rofi-search/blob/master/rofi-search.gif)](https://raw.githubusercontent.com/fogine/rofi-search/master/rofi-search.gif)
[(click for large preview)](https://raw.githubusercontent.com/fogine/rofi-search/master/rofi-search.gif)

Installation
-------------------
**AUR** package [rofi-search-git](https://aur.archlinux.org/packages/rofi-search-git/)  

or `npm install -g rofi-search`

or copy [rofi-search](/rofi-search) to your `$PATH`

Features
-------------------
* search as you type
* Google search
* DuckDuckGo search
* `I'm Feeling Lucky` - Open the first result directly without waiting for the results to render
* get top search results from multiple search engines
* copy search result website url
* open search result in web browser

Dependencies
-------------------
* `nodejs >= 8.2.1`
* [rofi-blocks](https://github.com/OmarCastro/rofi-blocks) (`rofi-blocks-git` package in AUR)
* [rofi](https://github.com/DaveDavenport/rofi/)
* `xclip` copy to clipboard
* [googler](https://github.com/jarun/googler) *(optional)*  google search results scraper
* [ddgr](https://github.com/jarun/ddgr) *(optional)*  DuckDuckGo search results scraper

HOW TO RUN
-------------------
Simply execute `rofi-search` in terminal (needs `googler` or `ddgr` dependency to be installed).  
Or see **Usage** section bellow for more advanced use cases.

Usage
------------------

You can choose between multiple methods of getting search results. Each one will give slightly different results.  
You can even let `rofi-search` combine search results from multiple search engines.

- Use [googler](https://github.com/jarun/googler) for scraping Google for search results
    - `googler` does not parse information about number of search results
        so this information is not currently available when using this method
- Use [ddgr](https://github.com/jarun/ddgr) for scraping DuckDuckGo for search results
    - `ddgr` does not parse information about number of search results
        so this information is not currently available when using this method
- Use google's custom search engine API by setting `GOOGLE_API_KEY` & `GOOGLE_SEARCH_ID` env variables

    - It can Search the entire web if you enable it in settings
    - You will need to go to [https://cse.google.com/cse/all](https://cse.google.com/cse/all) and create your own google custom search engine.
        - Get `Search engine ID` from the settings panel  
         ![Preview](https://github.com/fogine/rofi-search/blob/master/search_engine_key.png)

    - [Get API KEY](https://developers.google.com/custom-search/v1/introduction#identify_your_application_to_google_with_api_key) for the created search engine.
        ![Preview](https://github.com/fogine/rofi-search/blob/master/api_key.png)

#### Examples

##### Google CSE
```bash
export GOOGLE_API_KEY='google-api-key'
export GOOGLE_SEARCH_ID='google-search-engine-id'
export ROFI_SEARCH='cse'

rofi -modi blocks -blocks-wrap /absolute/path/to/rofi-search -show blocks \ 
-lines 4 -eh 4 -kb-custom-1 'Control+y' -theme /path/to/your/theme.rasi
``` 

##### googler
```bash
#for additional googler options see "googler --help"
export GOOGLE_ARGS='["--count", 5]'
export ROFI_SEARCH='googler'

rofi -modi blocks -blocks-wrap /absolute/path/to/rofi-search -show blocks \ 
-lines 4 -eh 4 -kb-custom-1 'Control+y' -theme /path/to/your/theme.rasi 
``` 

##### combine top free results from DuckDuckGo and Google
```bash
export DDG_ARGS='["-n", 3]'
export GOOGLE_ARGS='["--count", 3]'
export ROFI_SEARCH='googler,ddgr' #or 'cse,ddgr'

rofi -modi blocks -blocks-wrap /absolute/path/to/rofi-search -show blocks \ 
-lines 4 -eh 4 -kb-custom-1 'Control+y' -theme /path/to/your/theme.rasi
``` 

You can fetch rofi theme used in the gif preview [HERE](https://github.com/fogine/rofi-search/blob/master/theme.rasi)  


Options
-----------
- `GOOGLE_API_KEY` - secret key to access Google API. You can get it [here](https://developers.google.com/custom-search/v1/introduction#identify_your_application_to_google_with_api_key)
- `GOOGLE_SEARCH_ID` - Your custom search engine (cse) ID
- `GOOGLE_ARGS` - `googler` command line arguments. Serialized json array.
- `DDG_ARGS` - `ddgr` command line arguments. Serialized json array.
- `ROFI_SEARCH` - comma separated search methods
    - Supported methods: `cse`,`googler`,`ddgr`
    - If multiple methods are set, `rofi-search` will make multiple parallel searches
    and combine search results in the order search methods were defined
- `TITLE_COLOR` - customize search result title color (default blue).
                This can't be set in rofi theme
- `ROFI_SEARCH_TIMEOUT` - integer - delay between last character typed and automatic search execution (default `500`ms)
- `ROFI_SEARCH_DEBUG` - enables verbose logging if set to any value
- `ROFI_SEARCH_CMD` - a command to execute when pressing `-kb-accept-entry` (`enter`) - defaults to `xdg-open`
    - example: `export ROFI_SEARCH_CMD='google-chrome $URL'`

##### supported rofi actions

- `-kb-accept-entry` - open url with `xdg-open` (aka. your default browser)
- `-kb-accept-custom` - open search results on `google.com` in your browser
- `-kb-custom-1` - copy url to clipboard


#### TODO (PR is welcome)

- [x] ~~DuckDuckGO integration~~
- [ ] make font sizes configurable
- [ ] [buku](https://github.com/jarun/buku) integration
