# FWAC (FAA WJHTC Ansible CoP)

FWAC is the Ansible Community of Practice at the William J Hughes Technical Center in Egg Harbor New Jersey.

For information email [Scott Tully](mailto:scott.ctr.tully@faa.gov)

---

{% for post in site.posts %}
<article class="post">

  <h1><a href="{{ site.baseurl }}{{ post.url }}">{{ post.title }}</a></h1>

  <div class="entry">
    {{ post.excerpt }}
  </div>

  <a href="{{ site.baseurl }}{{ post.url }}" class="read-more">Read More</a>
  </article>
{% endfor %}
