<blockquote class="{% if class %}{{class}}{% endif %}">
  <p>
    {{ body | markdown(inline=true) }}
  </p>

{% if author %}
&mdash; {{author}}

{% if url %}
<a href="{{url}}">{{url_text}}</a>
{% endif %}
{% endif %}

</blockquote>
