# Posts

### Drafts

- [A Python API for FHIR Terminology](./posts/1-a-python-api-for-fhir-terminology)
- [7 Principles to build robust data pipelines](./posts/2-7-principles-for-data-pipelines)
- [5 rules to manage your dev teams credentials in a secure way](3-5-rules-to-manage-your-dev-teams-credentials-a-secure-way)

<ul>
  {% for post in site.posts %}
    <li>
      <a href="{{ post.url }}">{{ post.title }}</a>
      {{ post.excerpt }}
    </li>
  {% endfor %}
</ul>
