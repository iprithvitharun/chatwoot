<template>
  <div class="contacts-page row">
    <div class="left-wrap" :class="wrapClas">
      <contacts-header
        :search-query="searchQuery"
        :custom-views-id="customViewsId"
        :on-search-submit="onSearchSubmit"
        this-selected-contact-id=""
        :on-input-search="onInputSearch"
        :on-toggle-create="onToggleCreate"
        :on-toggle-import="onToggleImport"
        :on-toggle-filter="onToggleFilters"
        :header-title="pageTitle"
        @on-toggle-save-filter="onToggleSaveFilters"
      />
      <contacts-table
        :contacts="records"
        :show-search-empty-state="showEmptySearchResult"
        :is-loading="uiFlags.isFetching"
        :on-click-contact="openContactInfoPanel"
        :active-contact-id="selectedContactId"
        :sort-config="sortConfig"
        @on-sort-change="onSortChange"
      />
      <table-footer
        :on-page-change="onPageChange"
        :current-page="Number(meta.currentPage)"
        :total-count="meta.count"
      />
    </div>
    <add-custom-views
      v-if="showAddCustomViewsModal"
      :custom-views-query="customViewsQuery"
      :filter-type="filterType"
      @close="onCloseAddCustomViewsModal"
    />
    <contact-info-panel
      v-if="showContactViewPane"
      :contact="selectedContact"
      :on-close="closeContactInfoPanel"
    />
    <create-contact :show="showCreateModal" @cancel="onToggleCreate" />
    <woot-modal :show.sync="showImportModal" :on-close="onToggleImport">
      <import-contacts v-if="showImportModal" :on-close="onToggleImport" />
    </woot-modal>
    <woot-modal
      :show.sync="showFiltersModal"
      :on-close="onToggleFilters"
      size="medium"
    >
      <contacts-advanced-filters
        v-if="showFiltersModal"
        :on-close="onToggleFilters"
        :filter-types="contactFilterItems"
        @applyFilter="onApplyFilter"
        @clearFilters="clearFilters"
      />
    </woot-modal>
  </div>
</template>

<script>
import { mapGetters } from 'vuex';

import ContactsHeader from './Header';
import ContactsTable from './ContactsTable';
import ContactInfoPanel from './ContactInfoPanel';
import CreateContact from 'dashboard/routes/dashboard/conversation/contact/CreateContact';
import TableFooter from 'dashboard/components/widgets/TableFooter';
import ImportContacts from './ImportContacts.vue';
import ContactsAdvancedFilters from './ContactsAdvancedFilters.vue';
import contactFilterItems from '../contactFilterItems';
import filterQueryGenerator from '../../../../helper/filterQueryGenerator';
import AddCustomViews from 'dashboard/routes/dashboard/customviews/AddCustomViews';

const DEFAULT_PAGE = 1;
const FILTER_TYPE_CONTACT = 1;

export default {
  components: {
    ContactsHeader,
    ContactsTable,
    TableFooter,
    ContactInfoPanel,
    CreateContact,
    ImportContacts,
    ContactsAdvancedFilters,
    AddCustomViews,
  },
  props: {
    label: { type: String, default: '' },
    customViewsId: {
      type: [String, Number],
      default: 0,
    },
  },
  data() {
    return {
      searchQuery: '',
      showCreateModal: false,
      showImportModal: false,
      selectedContactId: '',
      sortConfig: { name: 'asc' },
      showFiltersModal: false,
      contactFilterItems: contactFilterItems.map(filter => ({
        ...filter,
        attributeName: this.$t(
          `CONTACTS_FILTER.ATTRIBUTES.${filter.attributeI18nKey}`
        ),
      })),
      customViewsQuery: {},
      filterType: FILTER_TYPE_CONTACT,
      showAddCustomViewsModal: false,
    };
  },
  computed: {
    ...mapGetters({
      records: 'contacts/getContacts',
      uiFlags: 'contacts/getUIFlags',
      meta: 'contacts/getMeta',
      customViews: 'customViews/getCustomViews',
      getAppliedContactFilters: 'contacts/getAppliedContactFilters',
    }),
    showEmptySearchResult() {
      const hasEmptyResults = !!this.searchQuery && this.records.length === 0;
      return hasEmptyResults;
    },
    hasAppliedFilters() {
      return this.getAppliedContactFilters.length;
    },
    hasActiveCustomViews() {
      return this.activeCustomView && this.customViewsId !== 0;
    },
    pageTitle() {
      if (this.hasActiveCustomViews) {
        return this.activeCustomView.name;
      }
      if (this.label) {
        return `#${this.label}`;
      }
      return this.$t('CONTACTS_PAGE.HEADER');
    },
    selectedContact() {
      if (this.selectedContactId) {
        const contact = this.records.find(
          item => this.selectedContactId === item.id
        );
        return contact;
      }
      return undefined;
    },
    showContactViewPane() {
      return this.selectedContactId !== '';
    },
    wrapClas() {
      return this.showContactViewPane ? 'medium-9' : 'medium-12';
    },
    pageParameter() {
      const selectedPageNumber = Number(this.$route.query?.page);
      return !Number.isNaN(selectedPageNumber) &&
        selectedPageNumber >= DEFAULT_PAGE
        ? selectedPageNumber
        : DEFAULT_PAGE;
    },
    activeCustomView() {
      if (this.customViewsId) {
        const [firstValue] = this.customViews.filter(
          view => view.id === Number(this.customViewsId)
        );
        return firstValue;
      }
      return undefined;
    },
  },
  watch: {
    label() {
      this.fetchContacts(DEFAULT_PAGE);
    },
    activeCustomView() {
      if (this.hasActiveCustomViews) {
        const payload = this.activeCustomView.query;
        this.fetchSavedFilteredContact(payload);
      } else {
        this.fetchContacts(DEFAULT_PAGE);
      }
    },
  },
  mounted() {
    this.fetchContacts(this.pageParameter);
  },
  methods: {
    updatePageParam(page) {
      window.history.pushState({}, null, `${this.$route.path}?page=${page}`);
    },
    getSortAttribute() {
      let sortAttr = Object.keys(this.sortConfig).reduce((acc, sortKey) => {
        const sortOrder = this.sortConfig[sortKey];
        if (sortOrder) {
          const sortOrderSign = sortOrder === 'asc' ? '' : '-';
          return `${sortOrderSign}${sortKey}`;
        }
        return acc;
      }, '');
      if (!sortAttr) {
        this.sortConfig = { name: 'asc' };
        sortAttr = 'name';
      }
      return sortAttr;
    },
    fetchContacts(page) {
      this.updatePageParam(page);
      let value = '';
      if (this.searchQuery.charAt(0) === '+') {
        value = this.searchQuery.substring(1);
      } else {
        value = this.searchQuery;
      }
      const requestParams = {
        page,
        sortAttr: this.getSortAttribute(),
        label: this.label,
      };
      if (!value) {
        this.$store.dispatch('contacts/get', requestParams);
      } else {
        this.$store.dispatch('contacts/search', {
          search: value,
          ...requestParams,
        });
      }
    },
    fetchSavedFilteredContact(payload) {
      if (this.hasAppliedFilters) {
        this.clearFilters();
      }
      this.$store.dispatch('contacts/filter', {
        queryPayload: payload,
      });
    },
    onInputSearch(event) {
      const newQuery = event.target.value;
      const refetchAllContacts = !!this.searchQuery && newQuery === '';
      this.searchQuery = newQuery;
      if (refetchAllContacts) {
        this.fetchContacts(DEFAULT_PAGE);
      }
    },
    onSearchSubmit() {
      this.selectedContactId = '';
      if (this.searchQuery) {
        this.fetchContacts(DEFAULT_PAGE);
      }
    },
    onPageChange(page) {
      this.selectedContactId = '';
      this.fetchContacts(page);
    },
    openContactInfoPanel(contactId) {
      this.selectedContactId = contactId;
      this.showContactInfoPanelPane = true;
    },
    closeContactInfoPanel() {
      this.selectedContactId = '';
      this.showContactInfoPanelPane = false;
    },
    onToggleCreate() {
      this.showCreateModal = !this.showCreateModal;
    },
    onToggleSaveFilters() {
      this.showAddCustomViewsModal = true;
    },
    onCloseAddCustomViewsModal() {
      this.showAddCustomViewsModal = false;
    },
    onToggleImport() {
      this.showImportModal = !this.showImportModal;
    },
    onSortChange(params) {
      this.sortConfig = params;
      this.fetchContacts(this.meta.currentPage);
    },
    onToggleFilters() {
      this.showFiltersModal = !this.showFiltersModal;
    },
    onApplyFilter(payload) {
      this.closeContactInfoPanel();
      this.customViewsQuery = { payload };
      this.$store.dispatch('contacts/filter', {
        queryPayload: filterQueryGenerator(payload),
      });
      this.showFiltersModal = false;
    },
    clearFilters() {
      this.$store.dispatch('contacts/clearContactFilters');
      this.fetchContacts(this.pageParameter);
    },
  },
};
</script>

<style lang="scss" scoped>
.contacts-page {
  width: 100%;
}

.left-wrap {
  display: flex;
  flex-direction: column;
  height: 100%;
}
</style>
