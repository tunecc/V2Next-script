<script>
import {PageType} from "@v2next/core/types.ts"
import {computed, nextTick} from "vue";
import Setting from "./components/Modal/SettingModal.vue";
import eventBus from "./utils/eventBus.js";
import {CMD} from "./utils/type.js";
import PostDetail from "./components/PostDetail.vue";
import Base64Tooltip from "./components/Base64Tooltip.vue";
import Msg from './components/Msg.vue';
import Tooltip from "./components/Tooltip.vue";
import TagModal from "./components/Modal/TagModal.vue";
import MsgModal from "./components/Modal/MsgModal.vue";
import {decodeEmail} from "./utils/email-decode.js";
import BaseSwitch from "./components/BaseSwitch.vue";
import BaseLoading from "./components/BaseLoading.vue";
import NotificationModal from "./components/Modal/NotificationModal.vue";
import BaseButton from "./components/BaseButton.vue";
import {
  applyThemeMode,
  DefaultVal,
  functions,
  getDefaultPost,
  getStoredThemePreference,
  normalizeThemePreference,
  resolveThemeMode,
  THEME_CACHE_KEY,
  THEME_USER_KEY,
  subscribeSystemThemeChange,
} from "@v2next/core/core.ts";
import {Icon} from "@iconify/vue";
import dayjs from "dayjs";

export default {
  components: {
    Icon,
    BaseButton,
    NotificationModal,
    BaseLoading, BaseSwitch, MsgModal, TagModal, Tooltip, Setting, PostDetail, Base64Tooltip, Msg
  },
  provide() {
    return {
      isLogin: computed(() => this.isLogin),
      isNight: computed(() => this.isNight),
      pageType: computed(() => this.pageType),
      tags: computed(() => this.tags),
      show: computed(() => this.show),
      post: computed(() => this.current),
      config: computed(() => this.config),
      allReplyUsers: computed(() => {
        if (this.current?.replyList) {
          return Array.from(new Set(this.current?.replyList?.map(v => v.username) ?? []))
        }
        return []
      }),
      showConfig: this.showConfig
    }
  },
  data() {
    return {
      loading: window.pageType === PageType.Post,
      refreshLoading: false,
      loadMore: false,
      isLogin: !!window.user.username,
      pageType: window.pageType,
      isNight: window.isNight,
      stopMe: window.stopMe,//停止使用脚本
      show: false,
      current: getDefaultPost(),
      list: [],
      config: functions.clone(window.config),
      tags: window.user.tags,
      configModal: {
        show: false
      },
      notificationModal: {
        show: false,
        loading: false,
        list: '',
        total: 0,
      },
      previewModal: {
        show: false,
        src: ''
      },
      popConfirmModal: {
        show: false,
        title: '',
        id: ''
      },
      timer: -1,
      timer2: -1,
      pageInfo: {
        title: '',
        number: 0
      },
      calendar: {
        show: false,
        year: '',
        month: '',
        dayCount: 0,
        firstDayWeek: 0,
        select: '',
        currentDate: '',
        currentLabel: '',
        hotDateSlotReady: false,
      },
      _unsubSystemTheme: null,
      _onStorageTheme: null,
    }
  },
  computed: {
    targetUserTags() {
      return this.tags[window.targetUserName] ?? []
    },
    isList() {
      return [PageType.Home, PageType.Node].includes(this.pageType)
    },
    isPost() {
      return this.pageType === PageType.Post
    },
    isMember() {
      return this.pageType === PageType.Member
    },
    calendarYearList() {
      const nowYear = new Date().getUTCFullYear()
      return Array.from({length: nowYear - 2010 + 1}, (_, i) => nowYear - i)
    },
  },
  watch: {
    config: {
      handler(newVal, oldVal) {
        console.log('config', functions.clone(newVal).notice, functions.clone(oldVal).notice)
        const preference = normalizeThemePreference(newVal?.themeMode, 'system')
        if (newVal.themeMode !== preference) {
          newVal.themeMode = preference
        }
        const configStr = localStorage.getItem('v2ex-config')
        const configObj = configStr ? JSON.parse(configStr) : {}
        const userKey = window.user.username || 'default'
        configObj[userKey] = newVal
        const defaultConfig = configObj.default ?? {}
        defaultConfig.themeMode = preference
        configObj.default = defaultConfig
        localStorage.setItem('v2ex-config', JSON.stringify(configObj))
        localStorage.setItem(THEME_CACHE_KEY, preference)
        localStorage.setItem(THEME_USER_KEY, userKey)
        window.config = newVal
        window.parse.editNoteItem(window.user.configPrefix + JSON.stringify(window.config), window.user.configNoteId)
      },
      deep: true
    },
    tags(newVal) {
      window.user.tags = newVal
    },
    'config.viewType'(newVal) {
      if (!newVal) return
      if (newVal === 'card') {
        $('.post-item').each(function () {
          $(this).addClass('preview')
        })
      } else {
        $('.post-item').each(function () {
          $(this).removeClass('preview')
        })
      }
    },
    'config.themeMode': {
      handler() {
        this.applyThemeByConfig()
        this.updateThemeToggleIcon()
      },
      immediate: true
    },
    'pageInfo.number'(newVal) {
      clearInterval(this.timer2)
      if (newVal) {
        document.title = `(${this.pageInfo.number}) ` + this.pageInfo.title
        if (this.config.notice.whenNewNoticeGlimmer) {
          let c = 0
          this.timer2 = setInterval(() => {
            c++
            document.title = this.pageInfo.title
            if (c % 2 === 0) {
              document.title = `(${this.pageInfo.number}) ` + this.pageInfo.title
            }
          }, 1000)
        }
      } else {
        document.title = this.pageInfo.title
      }
    },
    show(newVal) {
      // console.log('modelValue', newVal, window.history.state)
      if (this.pageType === PageType.Post) return
      if (newVal) {
        document.body.style.overflow = 'hidden'
        if (!window.history.state) {
          // console.log('执行了pushState', this.post.href)
          window.history.pushState({}, 0, this.current.href);
        }
        nextTick(() => {
          this.pageInfo.title = document.title = this.current.title ?? 'V2EX'
        })
      } else {
        document.body.style.overflow = 'unset'
        this.pageInfo.title = document.title = 'V2EX'
        if (window.history.state) {
          // console.log('执行了back')
          window.history.back();
        }
      }
    }
  },
  created() {
    let that = this
    this.initEvent()
    window.cb = this.winCb
    if (!window.canParseV2exPage) return

    //A标签的
    $(document).on('click', 'a', this.clickA)
    //主题的
    $(document).on('click', '.post-item', function (e) {
      // console.log('click-post-item')
      if (e.currentTarget.getAttribute('script')) return
      if (that.stopMe) return true
      //只有预览时，才响应点击
      if (this.classList.contains('preview')) {
        //A标签，要么上面的on事件已经处理了，要么就是不需要处理
        //IMG是头像
        //toggle是切换按钮
        if (e.target.tagName !== 'A'
          &&
          e.target.tagName !== 'IMG'
          &&
          !e.target.classList.contains('toggle')
        ) {
          // console.log('点空白处', this)
          let id = this.dataset['id']
          let href = this.dataset['href']
          if (id) {
            that.clickPost(e, id, href)
          } else {
            if (href) location.href = href
          }
        }
      }
    })
    //展开或收起的点击事件
    $(document).on('click', '.toggle', (e) => {
      if (this.stopMe) return true
      let id = e.target.dataset['id']
      let itemDom = document.querySelector(`.id_${id}`)
      if (itemDom.classList.contains('preview')) {
        e.target.innerText = '预览'
        itemDom.classList.remove('preview')
      } else {
        if (this.config.viewType !== 'card') {
          let index = this.list.findIndex(v => v.id == id)
          if (index > -1) {
            e.target.innerText = '收起'
            itemDom.classList.add('preview')
          } else {
            e.target.innerText = '加载中'
            functions.getPostDetailByApi(id).then(res => {
              if (res.content_rendered) {
                res.href = itemDom.dataset['href']
                this.list.push(getDefaultPost(res))
                itemDom.classList.add('preview')
                e.target.innerText = '收起'
                functions.appendPostContent(res, itemDom)
              } else {
                e.target.innerText = '预览'
                eventBus.emit(CMD.SHOW_MSG, {type: 'warning', text: '主题暂无正文！'})
              }
            })
          }
        } else {
          e.target.innerText = '收起'
          itemDom.classList.add('preview')
        }
      }
    })

    window.onpopstate = (event) => {
      if (event.state) {
        if (!this.show) this.show = true
      } else {
        if (this.show) this.show = false
      }
    };

    if (this.config.notice.takeOverNoticePage) {
      window.deleteNotification = (nId, token) => {
        // console.log('deleteNotification', nId, token)
        let item = $("#n_" + nId)
        item.slideUp('fast');
        $.post({
          url: '/delete/notification/' + nId + '?once=' + token,
          success() {
            $.get({
              url: '/notifications/below/' + window.notificationBottom,
              success(data, status, request) {
                item.remove()
                $('#notifications').append(that.checkReplyItemType(data));
                window.notificationBottom = request.getResponseHeader('X-V2EX-New-Notification-Bottom');
              },
              error() {
                item.slideDown('fast');
              }
            })
          },
          error() {
            item.slideDown('fast');
          }
        })
      }
    }
  },
  mounted() {
    this.calendar.hotDateSlotReady = Boolean(document.querySelector('#current-hot-date-slot'))
    if (this.calendar.hotDateSlotReady && !this.calendar.currentLabel) {
      this.setCurrentHotDate(this.getHotListBaseDate().format('YYYY-M-D'), '0')
    }
    this.interceptThemeToggle()

    this._unsubSystemTheme = subscribeSystemThemeChange(() => {
      if (normalizeThemePreference(this.config?.themeMode, 'system') !== 'system') return
      this.isNight = applyThemeMode('system', false)
      this.updateThemeToggleIcon?.()
    })

    this._onStorageTheme = (e) => {
      if (!e.key || (e.key !== THEME_CACHE_KEY && e.key !== 'v2ex-config')) return
      const userKey = window.user?.username || 'default'
      const next = getStoredThemePreference(userKey)
      if (!next) return
      const preference = normalizeThemePreference(next, 'system')
      if (this.config.themeMode === preference) {
        this.isNight = applyThemeMode(preference, false)
        this.updateThemeToggleIcon?.()
        return
      }
      this.config.themeMode = preference
      // themeMode watch 会 apply；若 deep watch 触发 note 同步可接受
    }
    window.addEventListener('storage', this._onStorageTheme)
  },
  beforeUnmount() {
    // console.log('unmounted')
    clearInterval(this.timer)
    eventBus.clear()
    $(document).off('click', 'a', this.clickA)
    if (this._unsubSystemTheme) {
      this._unsubSystemTheme()
      this._unsubSystemTheme = null
    }
    if (this._onStorageTheme) {
      window.removeEventListener('storage', this._onStorageTheme)
      this._onStorageTheme = null
    }
  },
  methods: {
    applyThemeByConfig() {
      const preference = normalizeThemePreference(this.config?.themeMode, 'system')
      if (this.config.themeMode !== preference) {
        // 只纠正非法值；system 必须保留
        this.config.themeMode = preference
      }
      this.isNight = applyThemeMode(preference, false)
      // 不要 setItem resolved；cache 已在 config watch 写 preference
      localStorage.setItem(THEME_CACHE_KEY, preference)
    },
    toggleThemePreference() {
      const resolved = resolveThemeMode(this.config.themeMode, false)
      this.config.themeMode = resolved === 'dark' ? 'light' : 'dark'
    },
    interceptThemeToggle() {
      const originToggle = document.querySelector('.light-toggle')
      if (originToggle) {
        if (!originToggle.dataset.v2nextThemeBound) {
          originToggle.addEventListener('click', (e) => {
            e.preventDefault()
            e.stopPropagation()
            this.toggleThemePreference()
          })
          originToggle.dataset.v2nextThemeBound = '1'
        }
        originToggle.removeAttribute('href')
      }

      const container = document.querySelector('#site-header .tools, #Top .tools, .tools')
      if (!container) return

      let toggle = container.querySelector('.v2next-theme-toggle')
      if (!toggle) {
        toggle = document.createElement('button')
        toggle.type = 'button'
        toggle.className = 'v2next-theme-toggle'
        toggle.setAttribute('aria-label', '切换 V2Next 深浅色模式')
        toggle.addEventListener('click', (e) => {
          e.preventDefault()
          e.stopPropagation()
          this.toggleThemePreference()
        })

        const anchor = container.querySelector('a[href^="/member/"], a[href*="/member/"], .avatar, #avatar, .light-toggle')
        if (anchor?.insertAdjacentElement) {
          anchor.insertAdjacentElement('afterend', toggle)
        } else {
          container.prepend(toggle)
        }
      }

      originToggle?.classList.add('v2next-origin-theme-toggle-hidden')
      this.updateThemeToggleIcon()
    },
    updateThemeToggleIcon() {
      const toggle = document.querySelector('.v2next-theme-toggle')
      if (!toggle) return
      const preference = normalizeThemePreference(this.config?.themeMode, 'system')
      const resolved = resolveThemeMode(preference, false)
      const size = 20
      const icons = {
        light: `<svg xmlns="http://www.w3.org/2000/svg" width="${size}" height="${size}" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><circle cx="12" cy="12" r="5"/><line x1="12" y1="1" x2="12" y2="3"/><line x1="12" y1="21" x2="12" y2="23"/><line x1="4.22" y1="4.22" x2="5.64" y2="5.64"/><line x1="18.36" y1="18.36" x2="19.78" y2="19.78"/><line x1="1" y1="12" x2="3" y2="12"/><line x1="21" y1="12" x2="23" y2="12"/><line x1="4.22" y1="19.78" x2="5.64" y2="18.36"/><line x1="18.36" y1="5.64" x2="19.78" y2="4.22"/></svg>`,
        dark: `<svg xmlns="http://www.w3.org/2000/svg" width="${size}" height="${size}" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M21 12.79A9 9 0 1 1 11.21 3 7 7 0 0 0 21 12.79z"/></svg>`
      }
      const resolvedLabel = resolved === 'dark' ? '深色' : '浅色'
      const nextLabel = resolved === 'dark' ? '浅色' : '深色'
      toggle.innerHTML = icons[resolved]
      if (preference === 'system') {
        toggle.title = `当前：跟随系统（${resolvedLabel}），点击切换为${nextLabel}`
      } else {
        toggle.title = `当前：${resolvedLabel}模式，点击切换为${nextLabel}`
      }
    },
    getHotListBaseDate() {
      const now = new Date()
      return dayjs(new Date(now.getUTCFullYear(), now.getUTCMonth(), now.getUTCDate()))
    },
    createLocalDate(year, month, date = 1) {
      return dayjs(new Date(Number(year), Number(month), Number(date)))
    },
    setCurrentHotDate(day, rawDate = '') {
      const dateNum = Number(rawDate)
      if (dateNum === 0) {
        this.calendar.currentDate = day
        this.calendar.currentLabel = '今天最热'
      } else if (dateNum === -1) {
        this.calendar.currentDate = day
        this.calendar.currentLabel = '昨天最热'
      } else if (dateNum === -2) {
        this.calendar.currentDate = day
        this.calendar.currentLabel = '前天最热'
      } else if (dateNum === 3) {
        this.calendar.currentDate = '3d'
        this.calendar.currentLabel = '近 3 天最热'
      } else if (dateNum === 7) {
        this.calendar.currentDate = '7d'
        this.calendar.currentLabel = '近 7 天最热'
      } else if (dateNum === 30) {
        this.calendar.currentDate = '30d'
        this.calendar.currentLabel = '近 30 天最热'
      } else {
        const d = dayjs(day)
        if (d.isValid()) {
          this.calendar.currentDate = d.format('YYYY-M-D')
          this.calendar.currentLabel = d.format('YYYY-MM-DD')
          this.calendar.select = d.format('YYYY-M-D')
          this.calendar.year = d.year()
          this.calendar.month = d.month()
          this.syncCalendarMonthMeta(d)
        } else {
          this.calendar.currentDate = day
          this.calendar.currentLabel = day
        }
      }
    },
    initCalendar(date) {
      const targetDate = date || this.getHotListBaseDate()
      this.calendar.year = targetDate.year()
      this.calendar.month = targetDate.month()
      this.calendar.dayCount = targetDate.daysInMonth()
      this.calendar.firstDayWeek = targetDate.startOf('month').day()
      this.calendar.select = targetDate.format('YYYY-M-D')
    },
    syncCalendarMonthMeta(baseDate) {
      this.calendar.dayCount = baseDate.daysInMonth()
      this.calendar.firstDayWeek = baseDate.startOf('month').day()
    },
    updateCalendarByYearMonth() {
      let now = this.createLocalDate(this.calendar.year, this.calendar.month)
      this.syncCalendarMonthMeta(now)
      const selectedDate = dayjs(this.calendar.select)
      const day = selectedDate.isValid() ? selectedDate.date() : 1
      now = now.date(Math.min(day, now.daysInMonth()))
      this.calendar.select = now.format('YYYY-M-D')
    },
    selectCalendarDay(day) {
      if (day <= 0) return
      const d = this.createLocalDate(this.calendar.year, this.calendar.month, day)
      this.calendar.select = d.format('YYYY-M-D')
    },
    getMonthDayInfo(num) {
      let now = this.createLocalDate(this.calendar.year, this.calendar.month)
      if (num > 0) {
        now = now.add(1, 'month')
      } else {
        now = now.subtract(1, 'month')
      }
      this.calendar.year = now.year()
      this.calendar.month = now.month()
      this.syncCalendarMonthMeta(now)
      this.updateCalendarByYearMonth()
    },
    checkReplyItemType(val) {
      let d = $(val)
      let str = d.html()
      if (str.includes('提到了你') || str.includes('回复了你')) {
        d.addClass('reply')
      }
      if (str.includes('感谢了你')) {
        d.addClass('star')
      }
      if (str.includes('收藏了你')) {
        d.addClass('collect')
      }
      return d
    },
    async getUnreadMessagesCount() {
      const res = await fetch(`${location.origin}/mission`)
      const htmlText = await res.text()
      const $page = $(htmlText)
      const text = $page.find('#Rightbar a[href^="/notifications"]').text()

      if (text.includes('未读提醒')) {
        const countStr = text.match(/\d+/)?.at(0)

        if (countStr) {
          return Number(text.match(/\d+/)?.at(0))
        }
      } else {
        return 0
      }
      throw new Error('无法获取未读消息数量')
    },
    clickA(e) {
      let that = this
      //有script表示是脚本生成的a标签用于新开页面的
      if (e.currentTarget.getAttribute('script')) return
      if (that.stopMe) return true

      let {pageType} = functions.checkPageType(e.currentTarget)
      let {href, id, title} = functions.parseA(e.currentTarget)

      // console.log('pageType', pageType, href)
      switch (pageType) {
        case PageType.Post:
          if (id) {
            that.clickPost(e, id, href, title)
          }
          break
        case PageType.Node:
        case PageType.Home:
        case PageType.Changes:
          return
        case PageType.Hot:
          let date = e.currentTarget.search.replace("?", "")
          if (date === 'setting') {
            if (this.calendar.show) {
              $('#Rightbar > .sep20:first').css('height', 'var(--component-margin)')
            } else {
              $('#Rightbar > .sep20:first').css('height', 'unset')
              if (this.calendar.currentDate && dayjs(this.calendar.currentDate).isValid()) {
                this.initCalendar(dayjs(this.calendar.currentDate))
              } else {
                this.initCalendar()
              }
            }
            this.calendar.show = !this.calendar.show
            functions.stopEvent(e)
            return
          }
          let now = this.getHotListBaseDate()
          let day = ''
          switch (Number(date)) {
            case 0:
              day = now.format('YYYY-M-D')
              break
            case -1:
              day = now.subtract(1, 'day').format('YYYY-M-D')
              break
            case -2:
              day = now.subtract(2, 'day').format('YYYY-M-D')
              break
            case 3:
              day = '3d'
              break
            case 7:
              day = '7d'
              break
            case 30:
              day = '30d'
              break
            default:
              day = date
              if (dayjs(day).isSame(now, 'day')) {
                this.setCurrentHotDate(day, date)
                functions.stopEvent(e)
                return location.reload()
              }
          }
          if (day) {
            this.setCurrentHotDate(day, date)
            fetch(DefaultVal.hotUrl + day + '.json').then(async r => {
              if (!r.ok) throw new Error(`hotlist ${day} ${r.status}`)
              let r1 = await r.json()
              if (!Array.isArray(r1)) throw new Error(`hotlist ${day} invalid response`)
              $('.cell.item.post-item').remove()
              r1.reverse().map(v => {
                let s = `
<div class="cell item post-item id_${v.id}" style="" data-href="https://www.v2ex.com/t/${v.id}#reply${v.replyCount}">
    <table cellpadding="0" cellspacing="0" border="0" width="100%">
      <tbody>
      <tr>
        <td width="48" valign="top" align="center">
          <a href="/member/${v.username}">
            <img src="${v.avatar}" class="avatar"
                 border="0" align="default"
                 width="48"
                 style="width: 48px; max-height: 48px;"
                 alt="ice9191">
          </a>
        </td>
        <td width="10"></td>
        <td width="auto" valign="middle">
          <span class="item_title">
            <a href="https://www.v2ex.com/t/${v.id}#reply${v.replyCount}" class="topic-link" id="topic-link-${v.id}">${v.title}</a>
          </span>
          <div class="sep5"></div>
          <span class="topic_info">
            <div class="votes"></div>
            <a class="node" href="/go/${v.nodeUrl}">${v.nodeTitle}</a> &nbsp;•&nbsp;
            <strong><a href="/member/${v.username}">${v.username}</a></strong> &nbsp;•&nbsp;
            <span title="${v.lastReplyDate}">${v.lastReplyDateAgo}</span> &nbsp;•&nbsp; 最后回复来自
            <strong><a href="/member/${v.lastReplyUsername}">${v.lastReplyUsername}</a></strong>
          </span>
        </td>
        <td width="70" align="right" valign="middle" style="position: relative;">
          <a href="/t/${v.id}#reply${v.replyCount}" class="count_livid">${v.replyCount}</a>
          <div data-id="${v.id}" class="toggle">预览</div>
        </td>
      </tr>
      </tbody>
    </table>
  </div>
                `
                $('#app').after($(s))
              })
            }).catch(e => {
              eventBus.emit(CMD.SHOW_MSG, {type: 'error', text: '暂无点击日期的最热数据！'})
            })
          }
          functions.stopEvent(e)
          return
        default:
          //夜间模式切换
          if (e.currentTarget.href.includes('/settings/night/toggle')) {
            this.toggleThemePreference()
            functions.stopEvent(e)
            return
          }
          //清除最近记录
          if (e.currentTarget.href === location.origin + '/#;') return
          //未读提醒
          if (e.currentTarget.href.includes('/notifications')) {
            this.pageInfo.number = 0
            $('#money').parent().prev().replaceWith(`<a href="/notifications">0 未读提醒</a>`)

            if (this.config.notice.takeOverNoticePage) {
              this.notificationModal.loading = true
              this.notificationModal.show = true
              fetch(href).then(async r => {
                let htmlText = await r.text()
                let bodyText = htmlText.match(/<body[^>]*>([\s\S]+?)<\/body>/g)
                let res = htmlText.match(/var notificationBottom = ([\d]+);/)
                if (res && res[1]) {
                  window.notificationBottom = Number(res[1])
                  console.log(' window.notificationBottom', window.notificationBottom)
                }

                let body = $(bodyText[0])
                let list = body.find('#notifications')
                //给每条通知分类
                list.children().each(function () {
                  that.checkReplyItemType(this)
                })
                let h = list.html()
                //获取总提醒数量
                let d = body.find('#Main > .box > .header .fr .gray')
                if (d.length) {
                  this.notificationModal.total = d.text()
                }
                this.notificationModal.list = h
                let p = list.next()
                //给翻页按钮加上A标签，原生的是onclick事件，我拦截不到
                let tds = p.find('.button')
                // console.log('td', tds)
                tds.each(function () {
                  let href = this.getAttribute('onclick')
                  if (href) {
                    this.innerHTML = `<a href=${href.replace('location.href=', '')}>${this.innerHTML}</a>`
                    this.setAttribute('onclick', '')
                  }
                })
                this.notificationModal.pages = p.html()
                this.notificationModal.loading = false
              }).catch(e => {
                this.notificationModal.loading = false
              })
              functions.stopEvent(e)
              return
            }
          }

          // functions.stopEvent(e)
          // console.log('click-a', e.currentTarget.pathname, e.currentTarget)
          // return

          if (that.config.newTabOpen) {
            functions.stopEvent(e)
            functions.openNewTab(e.currentTarget.href, that.config.newTabOpenActive)
          }
          return
      }
    },
    async clickPost(e, id, href, title = '') {
      // id = '976890'
      if (id) {
        if (this.config.clickPostItemOpenDetail) {
          functions.stopEvent(e)
          let postItem = getDefaultPost()
          let index = this.list.findIndex(v => v.id == id)
          if (index > -1) {
            postItem = this.list[index]
          }
          if (!postItem.title) postItem.title = title ?? '加载中'
          // console.log('postItem', JSON.stringify(postItem))
          postItem.id = id
          postItem.href = href
          this.getPostDetail(postItem)
          return
        }
        if (this.config.newTabOpen) {
          functions.stopEvent(e)
          functions.openNewTab(`https://www.v2ex.com/t/${id}?p=1`, this.config.newTabOpenActive)
        }
      }
    },
    showPost() {
      this.show = true
      $(`#Wrapper #Main .box:lt(3)`).each(function () {
        $(this).hide()
      })
    },
    showConfig() {
      this.configModal.show = true
    },
    resetTitle() {
      let r = document.title.match(/\s?\(\d+\)\s?/)
      if (r && r.length) {
        this.pageInfo.title = document.title.replace(r[0], '')
      } else {
        this.pageInfo.title = document.title
      }
    },
    async getNotice(body) {
      if (!body) {
        let res = await fetch('/t')
        if (res.status === 200) {
          let htmlText = await res.text()
          let bodyText = htmlText.match(/<body[^>]*>([\s\S]+?)<\/body>/g)
          body = $(bodyText[0])
        }
      }
      let notify = body.find('a[href="/notifications"]')
      if (notify.length) {
        this.resetTitle()
        let text = notify.text();
        if (text !== '0 未读提醒') {
          this.pageInfo.number = text.replace(' 未读提醒', '')
          console.log('text', text, this.config.notice.ddWebhook)
          if (this.config.notice.text !== text) {
            console.log('有新消息', text, this.config.notice.text,)
            $('#money').parent().prev().replaceWith(`<div><div class="orange-dot"></div><strong><a href="/notifications">${text}</a></strong></div>`)
            this.config.notice.text = text
            if (this.config.notice.ddWebhook) {
              let n = new Date()
              let s = n.getSeconds();
              s = (s < 10 ? "0" + s : s)
              let m = n.getMinutes()
              m = (m < 10 ? "0" + m : m)
              let h = n.getHours()
              h = (h < 10 ? "0" + h : h)
              $.ajax('https://car-back.ttentau.top/index.php/v1/config/forward', {
                method: 'POST',
                contentType: "application/json",
                data: JSON.stringify({
                  url: this.config.notice.ddWebhook,
                  "text": notify.text() + `，时间：${n.getFullYear()}/${n.getMonth() + 1}/${n.getDate()} ${h}:${m}:${s}`
                })
              })
            }
          }
        } else {
          $('#money').parent().prev().replaceWith(`<a href="/notifications">${text}</a>`)
          // console.log('消息清空',)
          this.config.notice.text = ''
        }
      }
    },
    async winCb({type, value}) {
      console.log('回调的类型', type, value)
      if (type === 'openSetting') {
        this.showConfig()
      }
      if (type === 'syncData') {
        this.stopMe = window.stopMe
      }
      if (type === 'getConfigSuccess') {
        if (window.config.version < DefaultVal.currentVersion && window.isDeadline) {
          $('.v2next-setting span').after(`<div class="new v2next-new">new</div>`)
        }
        if (window.isLogin && window.config.notice.loopCheckNotice) {
          this.getNotice($(document.body))
          this.timer = setInterval(this.getNotice, 1000 * 60 * Number(window.config.notice.loopCheckNoticeInterval))
        }

        this.config = window.config
        this.tags = window.user.tags
      }

      if (type === 'syncList') {
        this.list = Object.assign(this.list, window.postList)
      }

      if (type === 'warningNotice') {
        eventBus.emit(CMD.SHOW_MSG, {type: 'warning', text: value})
      }

      if (this.stopMe) return

      if (type === 'restorePost') {
        this.show = false
        this.loading = false
        eventBus.emit(CMD.SHOW_MSG, {type: 'warning', text: '脚本无法查看此页面！'})
        $(`#Wrapper #Main .box:lt(3)`).each(function () {
          $(this).show()
        })
      }

      if (type === 'postContent') {
        this.current = Object.assign(this.current, value)
        this.current.inList = true
        //这时有正文了，再打开，体验比较好
        if (this.config.autoOpenDetail) {
          this.showPost()
        }
      }

      if (type === 'postReplies') {
        this.loading = false
        this.current = Object.assign(this.current, value)
        // console.log('当前主题', functions.clone(this.current))
        this.list.push(functions.clone(this.current))
      }
    },
    regenerateReplyList() {
      // console.log('重新生成列表')
      if (this.current.replyList.length) {
        functions.createList(this.current, this.current.replyList)
      } else {
        this.current.replyCount = 0
        this.current.nestedReplies = []
        this.current.nestedRedundReplies = []
      }
      if (this.list.length) {
        let rIndex = this.list.findIndex(i => i.id === this.current.id)
        if (rIndex > -1) {
          this.list[rIndex] = functions.clone(this.current)
        }
      }
    },
    initEvent() {
      eventBus.on(CMD.CHANGE_COMMENT_THANK, (val) => {
        // console.log('CHANGE_COMMENT_THANK', val)
        const {id, type} = val
        let currentI = this.current.replyList.findIndex(i => i.id === id)
        if (currentI > -1) {
          this.current.replyList[currentI].isThanked = type === 'add'
          if (type === 'add') {
            this.current.replyList[currentI].thankCount++
          } else {
            this.current.replyList[currentI].thankCount--
          }
          this.regenerateReplyList()
        }
      })
      eventBus.on(CMD.CHANGE_POST_THANK, (val) => {
        const {id, type} = val
        this.current.isThanked = type === 'add'
        if (type === 'add') {
          this.current.thankCount++
        } else {
          this.current.thankCount--
        }
        let currentI = this.list.findIndex(i => i.id === id)
        if (currentI > -1) {
          this.list[currentI].isThanked = type === 'add'
          if (type === 'add') {
            this.list[currentI].thankCount++
          } else {
            this.list[currentI].thankCount++
          }
        }
      })
      eventBus.on(CMD.REMOVE, (val) => {
        // console.log('remove', val)
        let removeIndex = this.current.replyList.findIndex(i => i.floor === val)
        // console.log('removeIndex',removeIndex)
        if (removeIndex > -1) {
          this.current.replyList.splice(removeIndex, 1)
        }
        // console.log('removeIndex',this.current.replyList)

        this.regenerateReplyList()
        // this.msgList.push({...val, id: Date.now()})
      })
      eventBus.on(CMD.IGNORE, () => {
        this.show = false
        let rIndex = this.list.findIndex(i => i.id === this.current.id)
        if (rIndex > -1) {
          this.list.splice(rIndex, 1)
        }
        this.current = getDefaultPost()
      })
      eventBus.on(CMD.MERGE, (val) => {
        this.current = Object.assign(this.current, val)
        let rIndex = this.list.findIndex(i => i.id === this.current.id)
        if (rIndex > -1) {
          this.list[rIndex] = functions.clone(this.current)
        }
      })
      eventBus.on(CMD.ADD_REPLY, (item) => {
        this.current.replyList.push(item)
        this.regenerateReplyList()
      })
      eventBus.on(CMD.REFRESH_ONCE, async (once) => {
        if (once) {
          if (typeof once === 'string') {
            let res = once.match(/var once = "([\d]+)";/)
            if (res && res[1]) {
              this.current.once = Number(res[1])
              // console.log('接口返回了once-str', this.current.once)
              return
            }
          }
          if (typeof once === 'number') {
            this.current.once = once
            // console.log('接口返回了once-number', this.current.once)
            return
          }
        }
        window.fetchOnce().then(r => {
          // console.log('通过fetchOnce接口拿once', r)
          this.current.once = r
        })
      })
      eventBus.on(CMD.REMOVE_TAG, async ({username, tag}) => {
        let oldTag = functions.clone(this.tags)
        let tags = this.tags[username] ?? []
        let rIndex = tags.findIndex(v => v === tag)
        if (rIndex > -1) {
          tags.splice(rIndex, 1)
        }
        this.tags[username] = tags

        let res = await window.parse.saveTags(this.tags)
        if (!res) {
          eventBus.emit(CMD.SHOW_MSG, {type: 'error', text: '标签删除失败！'})
          this.tags = oldTag
        }
      })
      eventBus.on(CMD.SHOW_CONFIRM_MODAL, (val) => {
        const {rect, title, id} = val
        this.popConfirmModal.show = true
        this.popConfirmModal.title = title
        this.popConfirmModal.id = id
        nextTick(() => {
          this.$refs.tip.style.top = rect.top + 'px'
          this.$refs.tip.style.left = rect.left + rect.width / 2 - 50 + 'px'
        })
      })
    },
    async getPostDetail(post) {
      // console.log('getPostDetail')
      this.current = post
      this.show = true
      let url = location.origin + '/t/' + this.current.id
      this.current.url = url

      let alreadyHasReply = this.current.replyList.length
      //如果有数据，显示右侧的loading
      if (alreadyHasReply) {
        this.refreshLoading = true
      } else {
        this.loading = true

        functions.getPostDetailByApi(this.current.id).then(d => {
          d.replyCount = d.replies
          this.current = Object.assign(this.current, d)
          if (this.current.replyCount > window.config.maxReplyCountLimit) {
            functions.openNewTab(`${location.origin}/t/${this.current.id}?p=1&script=1`, true)
            eventBus.emit(CMD.SHOW_MSG, {type: 'warning', text: '由于回复数量较多，已为您单独打开此主题'})
            this.loading = this.show = false
            return
          } else {
            this.current.jsonContent = `
            <div class="cell">
              <div class="topic_content">
                <div class="markdown_body">
                 ${d?.content_rendered ?? ''}
                </div>
              </div>
            </div>`
          }
        })
      }

      //ajax不能判断是否跳转
      // $.get(url + '?p=1').then((res, textStatus, xhr) => {
      let apiRes = await window.fetch(url + '?p=1')
      if (apiRes.status === 404) {
        eventBus.emit(CMD.SHOW_MSG, {type: 'error', text: '主题未找到'})
        return this.refreshLoading = this.loading = false
      }
      if (apiRes.status === 403) {
        this.refreshLoading = this.show = this.loading = false
        functions.openNewTab(`${location.origin}/t/${post.id}?p=1&script=0`, true)
        return
      }
      //如果是重定向了，那么就是没权限
      if (apiRes.redirected) {
        eventBus.emit(CMD.SHOW_MSG, {type: 'error', text: '没有权限'})
        return this.refreshLoading = this.loading = false
      }
      let htmlText = await apiRes.text()
      let hasPermission = htmlText.search('你要查看的页面需要先登录')
      if (hasPermission > -1) {
        eventBus.emit(CMD.SHOW_MSG, {type: 'error', text: '你要查看的页面需要先登录'})
        return this.refreshLoading = this.loading = false
      }

      let bodyText = htmlText.match(/<body[^>]*>([\s\S]+?)<\/body>/g)
      let body = $(bodyText[0])

      decodeEmail(body)

      await window.parse.getPostDetail(this.current, body, htmlText)
      let index = this.list.findIndex(v => v.id == this.current.id)
      if (index > -1) {
        this.list[index] = functions.clone(this.current)
      } else {
        this.list.push(functions.clone(this.current))
      }
      this.refreshLoading = this.loading = false

      await window.parse.parseOp(this.current)

      console.log('当前主题', this.current)
    },
    addTargetUserTag() {
      eventBus.emit(CMD.ADD_TAG, window.targetUserName)
    },
    removeTargetUserTag(tag) {
      eventBus.emit(CMD.REMOVE_TAG, {username: window.targetUserName, tag})
    },
    popConfirmModalCancel() {
      this.popConfirmModal.show = false
    },
    popConfirmModalConfirm() {
      this.popConfirmModalCancel()
      eventBus.emit(CMD.SHOW_CONFIRM_MODAL_CONFIRM, this.popConfirmModal.id)
    },
  },
}
</script>

<template>
  <Setting v-model="config" v-model:show="configModal.show"/>
  <TagModal v-model:tags="tags"/>
  <PostDetail v-model="show"
              ref="postDetail"
              v-model:displayType="config.commentDisplayType"
              @refresh="getPostDetail(current)"
              :loading="loading"
              :refreshLoading="refreshLoading"
  />
  <Base64Tooltip/>
  <MsgModal/>
  <teleport to="#current-hot-date-slot" v-if="calendar.hotDateSlotReady && calendar.currentLabel">
    <div class="current-hot-date">
      <span class="dot"></span>
      <span class="label">当前</span>
      <span class="value">{{ calendar.currentLabel }}</span>
    </div>
  </teleport>
  <teleport to="#Rightbar > .sep20">
    <div class="" v-if="calendar.show">
      <div class="sep"></div>
      <div class="box calender">
        <div class="month">
          <div class="ca-title">
            <i class="fa fa-arrow-left"
               @click="getMonthDayInfo(-1)"
               aria-hidden="true"></i>
            <select v-model.number="calendar.year" @change="updateCalendarByYearMonth">
              <option v-for="y in calendarYearList" :key="y" :value="y">{{ y }}年</option>
            </select>
            <select v-model.number="calendar.month" @change="updateCalendarByYearMonth">
              <option v-for="m in 12" :key="m" :value="m - 1">{{ m }}月</option>
            </select>
            <i class="fa fa-arrow-right"
               @click="getMonthDayInfo(1)"
               aria-hidden="true"></i>
          </div>
        </div>
        <div class="calender-header">
          <div>日</div>
          <div>一</div>
          <div>二</div>
          <div>三</div>
          <div>四</div>
          <div>五</div>
          <div>六</div>
        </div>
        <div class="days">
          <div :class="[
            'day',
            calendar.select === `${calendar.year}-${calendar.month+1}-${i - calendar.firstDayWeek}`?'active':''
          ]"
               @click="selectCalendarDay(i - calendar.firstDayWeek)"
               v-for="i in calendar.dayCount+calendar.firstDayWeek">
            <a v-if="i - calendar.firstDayWeek > 0"
               :href="`/v2hot?${calendar.year}-${calendar.month+1}-${i - calendar.firstDayWeek}`">
              {{ i - calendar.firstDayWeek > 0 ? i - calendar.firstDayWeek : '' }}</a>
          </div>
        </div>
      </div>
      <div class="sep"></div>
    </div>
  </teleport>

  <NotificationModal
    v-model="notificationModal.show"
    :list="notificationModal.list"
    :loading="notificationModal.loading"
    :total="notificationModal.total"
    :pages="notificationModal.pages"
  />

  <template v-if="!stopMe">
    <div class="target-user-tags p1" v-if="isMember && isLogin && config.openTag">
      <span>标签：</span>
      <span class="my-tag" v-for="i in targetUserTags">
              <i class="fa fa-tag"></i>
              <span>{{ i }}</span>
              <i class="fa fa-trash-o remove" @click="removeTargetUserTag(i)"></i>
            </span>
      <span class="add-tag ago" @click="addTargetUserTag" title="添加标签">+</span>
    </div>
    <div v-if="isPost && !show " class="my-box p2" style="margin-top: 2rem;margin-bottom: 0;">
      <div class="flex flex-center" v-if="loading">
        <BaseLoading/>
      </div>
      <div v-else class="loaded">
        <span>楼中楼解析完成</span>
        <BaseButton size="small" @click="showPost">点击显示</BaseButton>
      </div>
    </div>
  </template>

  <Teleport to="body">
    <Transition>
      <div ref="tip" class="pop-confirm-content" v-if="popConfirmModal.show">
        <div class="text">
          {{ popConfirmModal.title }}
        </div>
        <div class="options">
          <BaseButton type="link" size="small" @click.stop="popConfirmModalCancel">取消</BaseButton>
          <BaseButton size="small" @click.stop="popConfirmModalConfirm">确认</BaseButton>
        </div>
      </div>
    </Transition>
  </Teleport>
</template>

<style lang="less">
@import "assets/less/index";
</style>
<style scoped lang="less">

.target-user-tags {
  background: var(--color-second-bg);
  color: var(--color-font);
  word-break: break-all;
  text-align: start;
  font-size: 1.4rem;
  box-shadow: 0 2px 3px rgba(0, 0, 0, .1);
  border-bottom-left-radius: 3px;
  border-bottom-right-radius: 3px;

  .add-tag {
    display: inline-block;
  }
}

.loaded {
  font-size: 1.4rem;
  display: flex;
  align-items: center;
  gap: 1rem;
  color: var(--color-font-pure);
}

.current-hot-date {
  margin: 0;
  display: inline-flex;
  align-items: center;
  border-radius: 1rem;
  padding: .2rem 1rem .2rem .6rem;
  background: linear-gradient(135deg, #e8f4fd, #dbeafe);
  gap: .5rem;
  min-height: 2rem;
  vertical-align: middle;
  border: 1px solid rgba(22, 119, 255, 0.15);
  position: relative;
  top: -1px;

  .dot {
    width: .6rem;
    height: .6rem;
    border-radius: 50%;
    background: #1677ff;
    animation: pulse-dot 2s ease-in-out infinite;
    flex-shrink: 0;
  }

  .label {
    color: #64748b;
    font-size: 1.1rem;
    font-weight: 500;
  }

  .value {
    color: #1677ff;
    font-size: 1.2rem;
    font-weight: 700;
  }
}

html.dark .current-hot-date {
  background: linear-gradient(135deg, #1e3a5f, #1a2f4a);
  border-color: rgba(64, 158, 255, 0.3);

  .dot {
    background: #409eff;
    box-shadow: 0 0 4px rgba(64, 158, 255, 0.6);
  }

  .label {
    color: rgba(255, 255, 255, 0.5);
  }

  .value {
    color: #409eff;
  }
}

@keyframes pulse-dot {
  0%, 100% { opacity: 1; transform: scale(1); }
  50% { opacity: 0.4; transform: scale(0.8); }
}

.calender {
  padding: 10px;
  font-size: 14px;
  color: var(--link-color);

  .month {
    height: 30px;
    display: flex;
    justify-content: space-between;
    align-items: center;

    .ca-title {
      flex: 1;
      display: flex;
      justify-content: flex-end;
      align-items: center;
      gap: 10px;
    }

    i {
      height: 100%;
      width: 30px;
      cursor: pointer;
      color: darkgrey;
    }

    select {
      height: 26px;
      border: 1px solid var(--color-input-border);
      border-radius: 4px;
      background: var(--color-input-bg);
      color: var(--color-font-8);
      outline: none;
    }
  }

  .calender-header {
    display: flex;
    height: 30px;
    align-items: center;

    div {
      flex: 1;
    }
  }

  .days {
    display: grid;
    grid-template-columns: repeat(7, 1fr);

    .day {
      height: 30px;

      a {
        display: inline-flex;
        height: 100%;
        width: 100%;
        justify-content: center;
        align-items: center;
      }
    }

    .active {
      background: #40a9ff;
      border-radius: 4px;

      a {
        color: white !important;
      }
    }
  }
}
</style>
