<template>
  <main>
    <project-meta
      :meta="page.meta"
      :views="page.meta.views"
      :view="view"
      :count="page.slides.length"
      @toggleView="toggleView($event)"
      @shuffle="shuffle" />

    <project-slides
      :meta="page.meta"
      :slides="page.slides"
      :view="view"
      @open="toggleView('slides', $event)"
      @close="close" />
  </main>
</template>

<script>
  import matter from 'gray-matter';
  import shuffle from 'lodash/shuffle';
  import ProjectMeta from '~/components/project/ProjectMeta.vue';
  import ProjectSlides from '~/components/project/ProjectSlides.vue';
  import path from 'path';

  export default {
    components: {
      ProjectMeta,
      ProjectSlides,
    },
    props: {
      src: {
        type: String,
        required: true,
      },
      demos: {
        type: Object,
        default: function () { return {} },
      },
    },
    data() {
      const page = this.parse(this.src);

      return {
        page: page,
        view: this.getView(page.meta.view),
        lastView: false,
      };
    },
    head() {
      const page = this.page.meta;
      const meta = [
        {
          hid: 'og_title', property: 'og:title',
          content: `${page.title}: ${page.subtitle}`,
        },
        {
          hid: 'og_url', property: 'og:url',
          content: `https://${this.projectUrl()}`,
        },
        {
          hid: 'og_type', property: 'og:type',
          content: 'article',
        },
        {
          hid: 'description', name: 'description',
          content: this.page.excerpt || this.page.content,
        },
        {
          property: 'article:published_time',
          content: `${page.date}T00:00:00`,
        },
      ];

      if (page.image) {
        meta.push(
          {
            hid: 'og_image', property: 'og:image',
            content: page.image,
          },
          {
            hid: 'twitter_card', property: 'twitter:card',
            content: 'summary_large_image',
          },
        );
      }

      if (page.author.facebook) {
        meta.push({
          property: 'article:author',
          content: `https://www.facebook.com/${page.author.facebook}`,
        });
      }

      if (page.author.twitter) {
        meta.push({
          property: 'twitter:creator',
          content: `@${page.author.twitter}`,
        });
      }

      if (page.tags) {
        page.tags.forEach(tag => {
          meta.push({
            property: 'article:tag',
            content: tag,
          });
        });
      }

      return {
        title: `${page.title} | OddTalks`,
        meta: meta
      }
    },
    methods: {
      projectUrl() {
        let route = this.$route.path.endsWith('/')
          ? this.$route.path
          : this.$route.path + '/';
        return path.join(process.env.domain, route);
      },
      // Actions
      shuffle() {
        this.page.slides = shuffle(this.page.slides);
      },
      getView(dataView) {
        return this.$route.query.view || dataView;
      },
      toggleView(newView, slide=null) {
        if (this.page.meta.views.includes(newView)
          && (newView !== this.view)) {
          slide = slide || this.$route.query.active;
          this.lastView = this.view;
          this.view = newView;
          this.$router.push({
            path: this.$route.path,
            query: {
              view: newView,
              active: slide,
            },
          });
        }
      },
      close() {
        if (this.lastView) {
          this.toggleView(this.lastView);
        } else {
          this.nextView();
        }
      },
      nextView() {
        const views = this.page.meta.views;
        const next = (views.indexOf(this.view) + 1) % views.length;
        this.toggleView(views[next]);
      },
      // parser
      parse(src) {
        const data = {
          slides: [],
        };

        // parse
        src.split("\n<!-- slide -->\n").forEach(function (partRaw, index) {
          partRaw = partRaw.trimLeft();

          const part = matter(partRaw, { excerpt_separator: "<!-- more -->" });
          part.content = part.content.trimLeft();
          part.excerpt = part.excerpt.trimLeft();

          if (index === 0) {
            data.meta = part.data;
            data.content = part.content;
            data.excerpt = part.excerpt;
          } else {
            part.id = `slide-${index}`;
            data.slides.push(part);
          }
        });

        // defaults
        data.meta.views = data.meta.views || ['grid', 'list', 'slides'];
        data.meta.view = data.meta.view || data.meta.views[0];
        data.meta.type = data.meta.type || 'talks';
        data.meta.demos = this.demos;
        data.meta.listen = true;
        data.meta.projectUrl = this.projectUrl();

        const author = data.meta.author || 0;
        data.meta.author = process.env.authors[author];

        return data;
      },
    },
  }
</script>

