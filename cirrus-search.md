## CirrusSearch

CirrusSearch provides a search engine implementation for MediaWiki core, extending core's SearchEngine class.

SearchEngine implementations provided in core use database backends.

## Special:Search

### Search modes

 * fulltext
 * go - searches for exact title matches or very near match, else does a full search

### Search profiles

Special:Search supports a 'profile' option:

* default
* advanced
* all
* images

Extensions can define additional profiles, using the SpecialSearchProfiles hook. CirrusSearch does not use this hook, though Translate and a couple other extensions do.

### Near match

* Considers variants, converting the search term to all variants and searching for all of them.
* Looks for exact title match.
* Then tries variations of the search text, such as all lower case or capitalized.
* Then gives hooks a try (SearchGetNearMatch).
 * CirrusSearch implements this hook, trying ascii folding and checking redirects.  If $wgCirrusSearchAllFields['use'] is true, then cirrus does a multimatch query on the 'all_near_match' and 'all_near_match.asciifolding' fields. (which contain titles and redirects)  $wgCirrusSearchAllFields['use'] and $wgCirrusSearchAllFields['build'] are true, by default.
* And tries / does some other stuff (e.g. go to user page, if searching user namespace, even if the user page does not exist)

CirrusSearch returns FancyTitleResultsType (with details on how the search term matched)

Hooks:

* SearchGetNearMatchBefore
* SearchAfterNoDirectMatch
* SearchGetNearMatch

If we want to support labels in near match, Wikibase could:

* add labels to the all_near_match field
* implement the SearchGetNearMatch and search the labels field in ElasticSearch or the Terms table. Considering asciifolding would be useful and such normalization would be useful.

Problem is that labels are not unique. If there are multiple near matches, then a near match search via the search bar takes you to the Special:Search results page.

### Full-text search

A title (searchTitle) and a text (searchText) search are done, as part of full text search, and the results are combined.

It is up to each SearchEngine to actually implement title and text search, and both are optional to support. (if not supported, then the search methods return null)

SearchMysql supports both, doing a MATCH query on either the si_text or si_title fields. CirrusSearch only implements text search.

CirrusSearch returns FullTextResultsType, which may include highlighting and a text snippet.

Actual search logic is in Searcher::searchText and Searcher::search

* In Searcher::searchText, special search syntax (e.g. incategory) is extracted and turned into filters, which are applied to the search query.

Filters available:

* prefix
* prefer-recent
* local - restrict search to local wiki
* insource - optionally, using a regex pattern
* boost-templates
* hastemplate
* linksto
* incategory
* intitle

### Optional query parameters for search:

* cirrusSuppressSuggest
* cirrusSuppressTitleHighlight
* cirrusSuppressAltTitle
* cirrusSuppressSnippet
* cirrusHighlightDefaultSimilarity
* cirrusHighlightAltTitleWithPostings

### Search query

A boolean query is done, combining search of the all.plain (^1) and all(^0.5) fields, with search of the all_near_match (^2) field. The all field search uses AND operator and then query rewrite (top_terms_boost_1024) is done. There should at least be a match of either the all fields or all_near_match, if not matching both.

The all_near_match field contains titles and redirects. For Wikibase, we want to consider near matches for labels. We could include labels in the all_near_match field, though this probably involves putting all labels in all languages into the all_near_match field.  This might be okay and then take language into account during rescoring.

## Prefix search

ApiOpenSearch (used for the search box on desktop) and ApiQueryPrefixSearch (used on mobile, etc) use the completion suggester, if enabled, for main namespace searches.

The older, more general prefix search implementation in CirrusSearch:

* Shares code in Searcher::search with full text search
* prefix search weights are applied or optionally $wgCirrusSearchPrefixSearchStartsWithAnyWord can be set
* A different rescore profile is used for prefix search

## Config

Configurations that can be adjusted:

* wgCirrusSearchPreferRecentDefaultDecayPortion
* wgCirrusSearchPreferRecentDefaultHalfLife
* wgCirrusSearchPreferRecentUnspecifiedDecayPortion
* wgCirrusSearchRegexMaxDeterminizedStates
* wgCirrusSearchStemmedWeight
* wgCirrusSearchNearMatchWeight
* wgCirrusSearchAllFields
* wgCirrusSearchPhraseRescoreBoost
* wgCirrusSearchPhraseRescoreWindowSize
* wgCirrusSearchAllFieldsForRescore

## Debugging

Query parameters:

* cirrusDumpQuery
* cirrusDumpResult
* cirrusExplain

api modules:

* action=cirrusdump - for any page, to see how it is indexed
* action=cirrus-config-dump
* action=cirrus-mapping-dump
* action=cirrus-settings-dump

