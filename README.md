# blog_source
blog framework: hexo
plugin：yarn add hexo-filter-mermaid-diagrams
add code in footer
```
<script src='https://unpkg.com/mermaid@7.1.2/dist/mermaid.min.js'></script>
      <script>
        if (window.mermaid) {
            mermaid.initialize({theme: 'forest'});
       }
</script>
```
       

deploy flow
1. hexo clean
2. hexo g
3. hexo s
4. hexo deploy
