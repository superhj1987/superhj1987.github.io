---
layout: page
title: "分类"
comments: true
sharing: false
footer: false
---
<script type="text/javascript">
$(document).ready(function(){
	$("#nav-menu a").removeClass("current");
	$("#nav-menu .categories-nav").addClass("current");
});
</script>
<ul>
{% for item in site.categories %}
    <li><a href="/blog/categories/{{ item[0] }}/">{{ item[0] | capitalize }}</a> [ {{ item[1].size }} ]</li>
{% endfor %}
</ul>
