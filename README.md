# FWAC (FAA WJHTC Ansible CoP)

FWAC is the Ansible Community of Practice at the William J Hughes Technical Center in Egg Harbor New Jersey.

>We are in the early stages of defining our goals and mission. Our first meeting will be held September 13 2018. For information email [Scott Tully](mailto:scott.ctr.tully@faa.gov)

* <http://s3.fwac.us/slides/meeting1.pptx>
* <http://s3.fwac.us/slides/meeting1_intro.pptx>

  {% for post in site.posts %}
    <article class="post">

      <h1><a href="{{ site.baseurl }}{{ post.url }}">{{ post.title }}</a></h1>

      <div class="entry">
        {{ post.excerpt }}
      </div>

      <a href="{{ site.baseurl }}{{ post.url }}" class="read-more">Read More</a>
    </article>
  {% endfor %}
