<template>
  <modal-inner aria-label="Export with Pandoc">
    <div class="modal__content">
      <p>Please choose a format for your <b>Pandoc export</b>.</p>
      <form-entry label="Template">
        <select class="textfield" slot="field" v-model="selectedFormat" @keydown.enter="resolve()">
          <option value="asciidoc">AsciiDoc</option>
          <option value="context">ConTeXt</option>
          <option value="epub">EPUB</option>
          <option value="epub3">EPUB v3</option>
          <option value="latex">LaTeX</option>
          <option value="odt">OpenOffice</option>
          <option value="pdf">PDF</option>
          <option value="rst">reStructuredText</option>
          <option value="rtf">Rich Text Format</option>
          <option value="textile">Textile</option>
          <option value="docx">Word</option>
        </select>
      </form-entry>
    </div>
    <div class="modal__button-bar">
      <button class="button" @click="config.reject()">Cancel</button>
      <button class="button button--resolve" @click="resolve()">Ok</button>
    </div>
  </modal-inner>
</template>

<script>
import FileSaver from 'file-saver';
import sponsorSvc from '../../services/sponsorSvc';
import networkSvc from '../../services/networkSvc';
import editorSvc from '../../services/editorSvc';
import googleHelper from '../../services/providers/helpers/googleHelper';
import modalTemplate from './common/modalTemplate';

export default modalTemplate({
  computedLocalSettings: {
    selectedFormat: 'pandocExportFormat',
  },
  methods: {
    async resolve() {
      this.config.resolve();
      const currentFile = this.$store.getters['file/current'];
      const currentContent = this.$store.getters['content/current'];
      const { selectedFormat } = this;
      this.$store.dispatch('queue/enqueue', async () => {
        const [sponsorToken, token] = await Promise.all([
          Promise.resolve().then(() => {
            const tokenToRefresh = this.$store.getters['workspace/sponsorToken'];
            return tokenToRefresh && googleHelper.refreshToken(tokenToRefresh);
          }),
          sponsorSvc.getToken(),
        ]);

        try {
          const { body } = await networkSvc.request({
            method: 'POST',
            url: 'pandocExport',
            params: {
              token,
              idToken: sponsorToken && sponsorToken.idToken,
              format: selectedFormat,
              options: JSON.stringify(this.$store.getters['data/computedSettings'].pandoc),
              metadata: JSON.stringify(currentContent.properties),
            },
            body: JSON.stringify(editorSvc.getPandocAst()),
            blob: true,
            timeout: 60000,
          });
          FileSaver.saveAs(body, `${currentFile.name}.${selectedFormat}`);
        } catch (err) {
          if (err.status === 401) {
            this.$store.dispatch('modal/open', 'sponsorOnly');
          } else {
            console.error(err); // eslint-disable-line no-console
            this.$store.dispatch('notification/error', err);
          }
        }
      });
    },
  },
});
</script>
