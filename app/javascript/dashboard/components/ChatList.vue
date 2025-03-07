<template>
  <div class="conversations-list-wrap">
    <slot></slot>
    <div
      class="chat-list__top"
      :class="{ filter__applied: hasAppliedFiltersOrActiveCustomViews }"
    >
      <h1 class="page-title text-truncate" :title="pageTitle">
        {{ pageTitle }}
      </h1>

      <div class="filter--actions">
        <chat-filter
          v-if="!hasAppliedFiltersOrActiveCustomViews"
          @statusFilterChange="updateStatusType"
        />
        <div v-if="hasAppliedFilters && !hasActiveCustomViews">
          <woot-button
            v-tooltip.top-end="$t('FILTER.CUSTOM_VIEWS.ADD.SAVE_BUTTON')"
            size="tiny"
            variant="smooth"
            color-scheme="secondary"
            icon="save"
            @click="onClickOpenAddCustomViewsModal"
          />
          <woot-button
            v-tooltip.top-end="$t('FILTER.CLEAR_BUTTON_LABEL')"
            size="tiny"
            variant="smooth"
            color-scheme="alert"
            icon="dismiss-circle"
            @click="resetAndFetchData"
          />
        </div>
        <div v-if="hasActiveCustomViews">
          <woot-button
            v-tooltip.top-end="$t('FILTER.CUSTOM_VIEWS.DELETE.DELETE_BUTTON')"
            size="tiny"
            variant="smooth"
            color-scheme="alert"
            icon="delete"
            class="delete-custom-view__button"
            @click="onClickOpenDeleteCustomViewsModal"
          />
        </div>

        <woot-button
          v-else
          v-tooltip.top-end="$t('FILTER.TOOLTIP_LABEL')"
          variant="clear"
          color-scheme="secondary"
          icon="filter"
          size="small"
          class="btn-filter"
          @click="onToggleAdvanceFiltersModal"
        >
        </woot-button>
      </div>
    </div>

    <add-custom-views
      v-if="showAddCustomViewsModal"
      :custom-views-query="customViewsQuery"
      @close="onCloseAddCustomViewsModal"
    />

    <delete-custom-views
      v-if="showDeleteCustomViewsModal"
      :show-delete-popup.sync="showDeleteCustomViewsModal"
      :active-custom-view="activeCustomView"
      :custom-views-id="customViewsId"
      @close="onCloseDeleteCustomViewsModal"
    />

    <chat-type-tabs
      v-if="!hasAppliedFiltersOrActiveCustomViews"
      :items="assigneeTabItems"
      :active-tab="activeAssigneeTab"
      class="tab--chat-type"
      @chatTabChange="updateAssigneeTab"
    />

    <p v-if="!chatListLoading && !conversationList.length" class="content-box">
      {{ $t('CHAT_LIST.LIST.404') }}
    </p>

    <div ref="activeConversation" class="conversations-list">
      <conversation-card
        v-for="chat in conversationList"
        :key="chat.id"
        :active-label="label"
        :team-id="teamId"
        :custom-views-id="customViewsId"
        :chat="chat"
        :conversation-type="conversationType"
        :show-assignee="showAssigneeInConversationCard"
      />

      <div v-if="chatListLoading" class="text-center">
        <span class="spinner"></span>
      </div>

      <woot-button
        v-if="!hasCurrentPageEndReached && !chatListLoading"
        variant="clear"
        size="expanded"
        @click="loadMoreConversations"
      >
        {{ $t('CHAT_LIST.LOAD_MORE_CONVERSATIONS') }}
      </woot-button>

      <p
        v-if="
          conversationList.length &&
            hasCurrentPageEndReached &&
            !chatListLoading
        "
        class="text-center text-muted end-of-list-text"
      >
        {{ $t('CHAT_LIST.EOF') }}
      </p>
    </div>
    <woot-modal
      :show.sync="showAdvancedFilters"
      :on-close="onToggleAdvanceFiltersModal"
      size="medium"
    >
      <conversation-advanced-filter
        v-if="showAdvancedFilters"
        :filter-types="advancedFilterTypes"
        :on-close="onToggleAdvanceFiltersModal"
        @applyFilter="onApplyFilter"
      />
    </woot-modal>
  </div>
</template>

<script>
import { mapGetters } from 'vuex';

import ChatFilter from './widgets/conversation/ChatFilter';
import ConversationAdvancedFilter from './widgets/conversation/ConversationAdvancedFilter';
import ChatTypeTabs from './widgets/ChatTypeTabs';
import ConversationCard from './widgets/conversation/ConversationCard';
import timeMixin from '../mixins/time';
import eventListenerMixins from 'shared/mixins/eventListenerMixins';
import conversationMixin from '../mixins/conversations';
import wootConstants from '../constants';
import advancedFilterTypes from './widgets/conversation/advancedFilterItems';
import filterQueryGenerator from '../helper/filterQueryGenerator.js';
import AddCustomViews from 'dashboard/routes/dashboard/customviews/AddCustomViews';
import DeleteCustomViews from 'dashboard/routes/dashboard/customviews/DeleteCustomViews.vue';

import {
  hasPressedAltAndJKey,
  hasPressedAltAndKKey,
} from 'shared/helpers/KeyboardHelpers';

export default {
  components: {
    AddCustomViews,
    ChatTypeTabs,
    ConversationCard,
    ChatFilter,
    ConversationAdvancedFilter,
    DeleteCustomViews,
  },
  mixins: [timeMixin, conversationMixin, eventListenerMixins],
  props: {
    conversationInbox: {
      type: [String, Number],
      default: 0,
    },
    teamId: {
      type: [String, Number],
      default: 0,
    },
    label: {
      type: String,
      default: '',
    },
    conversationType: {
      type: String,
      default: '',
    },
    customViewsId: {
      type: [String, Number],
      default: 0,
    },
  },
  data() {
    return {
      activeAssigneeTab: wootConstants.ASSIGNEE_TYPE.ME,
      activeStatus: wootConstants.STATUS_TYPE.OPEN,
      showAdvancedFilters: false,
      advancedFilterTypes: advancedFilterTypes.map(filter => ({
        ...filter,
        attributeName: this.$t(`FILTER.ATTRIBUTES.${filter.attributeI18nKey}`),
      })),
      customViewsQuery: {},
      showAddCustomViewsModal: false,
      showDeleteCustomViewsModal: false,
    };
  },
  computed: {
    ...mapGetters({
      currentChat: 'getSelectedChat',
      chatLists: 'getAllConversations',
      mineChatsList: 'getMineChats',
      allChatList: 'getAllStatusChats',
      unAssignedChatsList: 'getUnAssignedChats',
      chatListLoading: 'getChatListLoadingStatus',
      currentUserID: 'getCurrentUserID',
      activeInbox: 'getSelectedInbox',
      conversationStats: 'conversationStats/getStats',
      appliedFilters: 'getAppliedConversationFilters',
      customViews: 'customViews/getCustomViews',
    }),
    hasAppliedFilters() {
      return this.appliedFilters.length;
    },
    hasActiveCustomViews() {
      return this.activeCustomView && this.customViewsId !== 0;
    },
    hasAppliedFiltersOrActiveCustomViews() {
      return this.hasAppliedFilters || this.hasActiveCustomViews;
    },
    savedCustomViewsValue() {
      if (this.hasActiveCustomViews) {
        const payload = this.activeCustomView.query;
        this.fetchSavedFilteredConversations(payload);
      }
      return {};
    },
    assigneeTabItems() {
      return this.$t('CHAT_LIST.ASSIGNEE_TYPE_TABS').map(item => {
        const count = this.conversationStats[item.COUNT_KEY] || 0;
        return {
          key: item.KEY,
          name: item.NAME,
          count,
        };
      });
    },
    showAssigneeInConversationCard() {
      return this.activeAssigneeTab === wootConstants.ASSIGNEE_TYPE.ALL;
    },
    inbox() {
      return this.$store.getters['inboxes/getInbox'](this.activeInbox);
    },
    currentPage() {
      return this.$store.getters['conversationPage/getCurrentPageFilter'](
        this.activeAssigneeTab
      );
    },
    currentPageFilterKey() {
      return this.hasAppliedFiltersOrActiveCustomViews
        ? 'appliedFilters'
        : this.activeAssigneeTab;
    },
    currentFiltersPage() {
      return this.$store.getters['conversationPage/getCurrentPageFilter'](
        this.currentPageFilterKey
      );
    },
    hasCurrentPageEndReached() {
      return this.$store.getters['conversationPage/getHasEndReached'](
        this.currentPageFilterKey
      );
    },
    conversationFilters() {
      return {
        inboxId: this.conversationInbox ? this.conversationInbox : undefined,
        assigneeType: this.activeAssigneeTab,
        status: this.activeStatus,
        page: this.currentPage + 1,
        labels: this.label ? [this.label] : undefined,
        teamId: this.teamId ? this.teamId : undefined,
        conversationType: this.conversationType
          ? this.conversationType
          : undefined,
        customViews: this.hasActiveCustomViews
          ? this.savedCustomViewsValue
          : undefined,
      };
    },
    pageTitle() {
      if (this.inbox.name) {
        return this.inbox.name;
      }
      if (this.activeTeam.name) {
        return this.activeTeam.name;
      }
      if (this.label) {
        return `#${this.label}`;
      }
      if (this.conversationType === 'mention') {
        return this.$t('CHAT_LIST.MENTION_HEADING');
      }
      if (this.hasActiveCustomViews) {
        return this.activeCustomView.name;
      }
      return this.$t('CHAT_LIST.TAB_HEADING');
    },
    conversationList() {
      let conversationList = [];
      if (!this.hasAppliedFiltersOrActiveCustomViews) {
        const filters = this.conversationFilters;
        if (this.activeAssigneeTab === 'me') {
          conversationList = [...this.mineChatsList(filters)];
        } else if (this.activeAssigneeTab === 'unassigned') {
          conversationList = [...this.unAssignedChatsList(filters)];
        } else {
          conversationList = [...this.allChatList(filters)];
        }
      } else {
        conversationList = [...this.chatLists];
      }

      return conversationList;
    },
    activeCustomView() {
      if (this.customViewsId) {
        const activeView = this.customViews.filter(
          view => view.id === Number(this.customViewsId)
        );
        const [firstValue] = activeView;
        return firstValue;
      }
      return undefined;
    },
    activeTeam() {
      if (this.teamId) {
        return this.$store.getters['teams/getTeam'](this.teamId);
      }
      return {};
    },
  },
  watch: {
    activeTeam() {
      this.resetAndFetchData();
    },
    conversationInbox() {
      this.resetAndFetchData();
    },
    label() {
      this.resetAndFetchData();
    },
    conversationType() {
      this.resetAndFetchData();
    },
    activeCustomView() {
      this.resetAndFetchData();
    },
  },
  mounted() {
    this.$store.dispatch('setChatFilter', this.activeStatus);
    this.resetAndFetchData();

    bus.$on('fetch_conversation_stats', () => {
      this.$store.dispatch('conversationStats/get', this.conversationFilters);
    });
  },
  methods: {
    onApplyFilter(payload) {
      if (this.$route.name !== 'home') {
        this.$router.push({ name: 'home' });
      }
      this.customViewsQuery = { payload: payload };
      this.$store.dispatch('conversationPage/reset');
      this.$store.dispatch('emptyAllConversations');
      this.fetchFilteredConversations(payload);
    },
    onClickOpenAddCustomViewsModal() {
      this.showAddCustomViewsModal = true;
    },
    onCloseAddCustomViewsModal() {
      this.showAddCustomViewsModal = false;
    },
    onClickOpenDeleteCustomViewsModal() {
      this.showDeleteCustomViewsModal = true;
    },
    onCloseDeleteCustomViewsModal() {
      this.showDeleteCustomViewsModal = false;
    },
    onToggleAdvanceFiltersModal() {
      this.showAdvancedFilters = !this.showAdvancedFilters;
    },
    getKeyboardListenerParams() {
      const allConversations = this.$refs.activeConversation.querySelectorAll(
        'div.conversations-list div.conversation'
      );
      const activeConversation = this.$refs.activeConversation.querySelector(
        'div.conversations-list div.conversation.active'
      );
      const activeConversationIndex = [...allConversations].indexOf(
        activeConversation
      );
      const lastConversationIndex = allConversations.length - 1;
      return {
        allConversations,
        activeConversation,
        activeConversationIndex,
        lastConversationIndex,
      };
    },
    handleKeyEvents(e) {
      if (hasPressedAltAndJKey(e)) {
        const {
          allConversations,
          activeConversationIndex,
        } = this.getKeyboardListenerParams();
        if (activeConversationIndex === -1) {
          allConversations[0].click();
        }
        if (activeConversationIndex >= 1) {
          allConversations[activeConversationIndex - 1].click();
        }
      }
      if (hasPressedAltAndKKey(e)) {
        const {
          allConversations,
          activeConversationIndex,
          lastConversationIndex,
        } = this.getKeyboardListenerParams();
        if (activeConversationIndex === -1) {
          allConversations[lastConversationIndex].click();
        } else if (activeConversationIndex < lastConversationIndex) {
          allConversations[activeConversationIndex + 1].click();
        }
      }
    },
    resetAndFetchData() {
      this.$store.dispatch('conversationPage/reset');
      this.$store.dispatch('emptyAllConversations');
      this.$store.dispatch('clearConversationFilters');
      if (this.hasActiveCustomViews) {
        const payload = this.activeCustomView.query;
        this.fetchSavedFilteredConversations(payload);
      }
      if (this.customViewsId) {
        return;
      }
      this.fetchConversations();
    },
    fetchConversations() {
      this.$store
        .dispatch('fetchAllConversations', this.conversationFilters)
        .then(() => this.$emit('conversation-load'));
    },
    loadMoreConversations() {
      if (!this.hasAppliedFiltersOrActiveCustomViews) {
        this.fetchConversations();
      }
      if (this.hasActiveCustomViews) {
        const payload = this.activeCustomView.query;
        this.fetchSavedFilteredConversations(payload);
      } else {
        this.fetchFilteredConversations(this.appliedFilters);
      }
    },
    fetchFilteredConversations(payload) {
      let page = this.currentFiltersPage + 1;
      this.$store
        .dispatch('fetchFilteredConversations', {
          queryData: filterQueryGenerator(payload),
          page,
        })
        .then(() => this.$emit('conversation-load'));
      this.showAdvancedFilters = false;
    },
    fetchSavedFilteredConversations(payload) {
      let page = this.currentFiltersPage + 1;
      this.$store
        .dispatch('fetchFilteredConversations', {
          queryData: payload,
          page,
        })
        .then(() => this.$emit('conversation-load'));
    },
    updateAssigneeTab(selectedTab) {
      if (this.activeAssigneeTab !== selectedTab) {
        bus.$emit('clearSearchInput');
        this.activeAssigneeTab = selectedTab;
        if (!this.currentPage) {
          this.fetchConversations();
        }
      }
    },
    updateStatusType(index) {
      if (this.activeStatus !== index) {
        this.activeStatus = index;
        this.resetAndFetchData();
      }
    },
  },
};
</script>

<style scoped lang="scss">
@import '~dashboard/assets/scss/woot';

.spinner {
  margin-top: var(--space-normal);
  margin-bottom: var(--space-normal);
}

.conversations-list-wrap {
  flex-shrink: 0;
  width: 34rem;

  @include breakpoint(large up) {
    width: 36rem;
  }
  @include breakpoint(xlarge up) {
    width: 35rem;
  }
  @include breakpoint(xxlarge up) {
    width: 38rem;
  }
  @include breakpoint(xxxlarge up) {
    flex-basis: 46rem;
  }
}
.filter--actions {
  display: flex;
  align-items: center;
}

.btn-filter {
  margin: 0 var(--space-smaller);
}

.filter__applied {
  padding: 0 0 var(--space-slab) 0 !important;
  border-bottom: 1px solid var(--color-border);
}

.delete-custom-view__button {
  margin-right: var(--space-normal);
}
</style>
