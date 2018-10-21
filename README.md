<!--
[![Maintainability](https://api.codeclimate.com/v1/badges/124348de2ee6850d682f/maintainability)](https://codeclimate.com/github/andrew-codes/gatsby-plugin-elasticlunr-search/maintainability)
[![Codacy Badge](https://api.codacy.com/project/badge/Grade/7230ae7191f44a9489834553760310c2)](https://www.codacy.com/app/andrew-codes/gatsby-plugin-elasticlunr-search?utm_source=github.com&amp;utm_medium=referral&amp;utm_content=andrew-codes/gatsby-plugin-elasticlunr-search&amp;utm_campaign=Badge_Grade)

-->

# Search Plugin for Gatsby

This plugin enables search integration via elastic lunr. Content is indexed and then made available via graphql to rehydrate into an `elasticlunr` index. From there, queries can be made against this index to retrieve pages by their ID.

It is a fork of [gatsby-plugin-elasticlunr-search](https://github.com/andrew-codes/gatsby-plugin-elasticlunr-search) made in order to use the plugin with gatsby-v2.

# Getting Started

Install the plugin via `npm install --save @gatsby-contrib/gatsby-plugin-elasticlunr-search`.

See the [example site](https://gatsby-contrib.github.io/gatsby-plugin-elasticlunr-search/) [code](./example) for more specific implementation details.

Next, update your `gatsby-config.js` file to utilize the plugin.

## Setup in `gatsby-config`

Here's an example for a site that create pages using markdown, in which you you'd like to allow search features for `title` and `tags` frontmatter entries.

`gatsby-config.js`
```javascript
module.exports = {
    plugins: [
        {
            resolve: `@gatsby-contrib/gatsby-plugin-elasticlunr-search`,
            options: {
                // Fields to index
                fields: [
                    `title`,
                    `tags`,
                ],
                // How to resolve each field`s value for a supported node type
                resolvers: {
                    // For any node of type MarkdownRemark, list how to resolve the fields` values
                    MarkdownRemark: {
                        title: node => node.frontmatter.title,
                        tags: node => node.frontmatter.tags,
                        path: node => node.frontmatter.path,
                    },
                },
            },
        },
    ],
};
```

## Consuming in Your Site

The serialized search index will be available via graphql. Once queried, a component can create a new elasticlunr index with the value retrieved from the graphql query. Search queries can be made against the hydrated search index. The results is an array of document IDs. The index can return the full document given a document ID.

In gatsby-v2, it is possible to use graphql queries inside components using [`StaticQuery`](https://www.gatsbyjs.org/docs/static-query/).

Suppose that you want to include the `Search` component inside an `Header` component. *(Of course, you could also query `siteSearchIndex` from `layout.js` component, and pass it down as prop to any component that need it.)*

First, query the data with `StaticQuery` inside the `Header` component, and pass it as props to the `Search` component.

`components/header.js`
```javascript
import React from 'react'
import { StaticQuery, Link } from 'gatsby'
import { graphql } from 'gatsby'

import Search from './search'

const Header = () => (
    <StaticQuery
        query={graphql`
            query SearchIndexQuery {
                siteSearchIndex {
                    index
                }
            }
        `}
        render={data => (
            <header>
                ... header stuff...
                <Search searchIndex={data.siteSearchIndex.index} />
            </header>
        )}
    />
)

export default Header
```

And then use the `searchIndex` inside your `Search` component.

`components/search.js`
```javascript
import React, { Component } from 'react'
import { Index } from 'elasticlunr'

// Search component
export default class Search extends Component {
    constructor(props) {
        super(props)
        this.state = {
            query: ``,
            results: [],
        }
    }

    render() {
        return (
            <div>
                <input
                    type="text"
                    value={this.state.query}
                    onChange={this.search}
                />
                <ul>
                    {this.state.results.map(page => (
                         <li key={page.id}>
                            <Link to={'/' + page.path}>
                                {page.title}
                            </Link>
                            {': ' + page.tags.join(`,`)}
                        </li>
                    ))}
                </ul>
            </div>
        )
    }
    getOrCreateIndex = () =>
        this.index
            ? this.index
            : // Create an elastic lunr index and hydrate with graphql query results
              Index.load(this.props.searchIndex)

    search = evt => {
        const query = evt.target.value
        this.index = this.getOrCreateIndex()
        this.setState({
            query,
            // Query the index with search string to get an [] of IDs
            results: this.index
                .search(query)
                // Map over each ID and return the full document
                .map(({ ref }) => this.index.documentStore.getDoc(ref)),
        })
    }
}
```
