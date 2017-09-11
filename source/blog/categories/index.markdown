---
layout: page
title: "分类"
comments: false
sharing: false
footer: false
---
<script type="text/javascript">
$(document).ready(function(){
	$("#nav-menu a").removeClass("current");
	$("#nav-menu .categories-nav").addClass("current");
});

</script>
<ul style="margin-left: -45px !important;margin-top:30px !important;">
{% for item in site.categories %}
    <li style="margin-bottom: 20px !important;margin-right:10px;font-size: 16px !important;width: 200px !important;display: inline-block !important;padding-left: 5px !important"><a href="/blog/categories/{{ item[0] }}/">{{ item[0] | capitalize }}</a> [ {{ item[1].size }} ]</li>
{% endfor %}
</ul>
