# rofi-google
Interactive Google search via [rofi](https://github.com/DaveDavenport/rofi/).  

[![Preview](a/raw/b/rofi-google.gif)](a/raw/b/rofi-google.gif)
[(click for large preview)](a/raw/b/rofi-google.gif)

Installation
-------------------
`npm install -g rofi-google`

or copy [rofi-google](a/raw/b/rofi-google) to your `$PATH`

Features
-------------------
* search as you type
* copy search result website url
* open search result in web browser

Dependencies
-------------------
* [rofi-blocks](https://github.com/OmarCastro/rofi-blocks) (currently it needs patched version that has fix for `utf-8` encoding, you can get it here [patched-rofi-blocks](https://github.com/fogine/rofi-blocks/tree/next))
* [rofi](https://github.com/DaveDavenport/rofi/)
* `xclip` copy to clipboard
* [googler](https://github.com/jarun/googler) *(optional)*  

Usage
------------------

You can choose between two methods of getting search results. Each one will give slightly different results.  

- Use [googler](https://github.com/jarun/googler) for scraping the web for search results (default behavior)
    - `googler` does not parse information about number of search results
    - slightly slower than the other method which uses official google API
- Use google's custom search engine API and set `GOOG_API_KEY` & `GOOG_SEARCH_ID` env variables

    - You will need to go to [https://cse.google.com/cse/all](https://cse.google.com/cse/all) and create your own google custom search engine.
        - Get `Search engine ID` from the settings panel  
         ![Preview](a/raw/b/search_engine_key.png)

    - [Get API KEY](https://developers.google.com/custom-search/v1/introduction#identify_your_application_to_google_with_api_key) for the created search engine.
        ![Preview](a/raw/b/api_key.png)


#### rofi

```bash
#either use custom search engine
export GOOG_API_KEY='google-api-key'
export GOOG_SEARCH_ID='google-search-engine-id'

#or optionaly set additional googler options (see googler --help)
export GOOGLER_ARGS='["--count", 5]'


#TITLE_COLOR allows you to customize search results title color (default blue),
#as this can't be set via rofi theme
export TITLE_COLOR='#3296c8'

rofi -modi blocks -blocks-wrap rofi-google -show blocks \ 
-lines 4 -eh 4 -kb-custom-1 'Control+y' -theme /path/to/your/theme.rasi
``` 

You can fetch rofi theme used in the gif preview [HERE](https://github.com/fogine/dotfiles/blob/master/rofi/google_theme.css)

##### supported actions

- `-kb-accept-entry` - open url with `xdg-open` (aka. your default browser)
- `-kb-accept-custom` - open search results on `google.com` in your browser
- `-kb-custom-1` - copy url to clipboard


#### TODO (PR is welcome)

- [ ] make font sizes configurable
