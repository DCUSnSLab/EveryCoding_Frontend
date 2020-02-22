<template>
  <Row type="flex">
    <Col :span="24">
    <Panel id="contest-card" shadow>
      <div slot="title">{{ $route.params.lectureTitle }}</div><!--LectureList.vue에서 보낸 수강과목 title 값-->
      <div slot="extra">
        <ul class="filter">
          <li>
            <Dropdown @on-click="onRuleChange">
              <span>{{query.rule_type === '' ? this.$i18n.t('m.Rule') : this.$i18n.t('m.' + query.rule_type)}}
                <Icon type="arrow-down-b"></Icon>
              </span>
              <Dropdown-menu slot="list">
                <Dropdown-item name="">{{$t('m.All')}}</Dropdown-item>
                <Dropdown-item name="OI">{{$t('m.OI')}}</Dropdown-item>
                <Dropdown-item name="ACM">{{$t('m.ACM')}}</Dropdown-item>
              </Dropdown-menu>
            </Dropdown>
          </li>
          <li>
            <Dropdown @on-click="onStatusChange">
              <span>{{query.status === '' ? this.$i18n.t('m.Status') : this.$i18n.t('m.' + CONTEST_STATUS_REVERSE[query.status].name.replace(/ /g,"_"))}}
                <Icon type="arrow-down-b"></Icon>
              </span>
              <Dropdown-menu slot="list">
                <Dropdown-item name="">{{$t('m.All')}}</Dropdown-item>
                <Dropdown-item name="0">{{$t('m.Underway')}}</Dropdown-item>
                <Dropdown-item name="1">{{$t('m.Not_Started')}}</Dropdown-item>
                <Dropdown-item name="-1">{{$t('m.Ended')}}</Dropdown-item>
              </Dropdown-menu>
            </Dropdown>
          </li>
          <li>
            <Input id="keyword" @on-enter="changeRoute" @on-click="changeRoute" v-model="query.keyword"
                   icon="ios-search-strong" placeholder="Keyword"/>
          </li>
        </ul>
      </div>
      <p id="no-contest" v-if="contests.length == 0">{{$t('m.No_contest')}}</p>
      <ol id="contest-list">
        <li v-for="contest in contests" :key="contest.title">
		<!--수강 과목에 포함된 contest만을 (contest에서 assigned lecture의 값이 lecture id 값과 같은 경우) contest를 출력한다.-->
            <Row type="flex" justify="space-between" align="middle">
            <img class="trophy" src="../../../../assets/Cup.png"/><!--다른 이미지로 변경 요망-->
            <Col :span="18" class="contest-main">
            <p class="title">
              <a class="entry" @click.stop="goContest(contest)">
                {{contest.title}}
              </a>
              <template v-if="contest.contest_type != 'Public'">
                <Icon type="ios-locked-outline" size="20"></Icon>
              </template>
            </p>
            <ul class="detail">
              <li>
                <Icon type="calendar" color="#3091f2"></Icon>
                {{contest.start_time | localtime('YYYY-M-D HH:mm') }}
              </li>
              <li>
                <Icon type="android-time" color="#3091f2"></Icon>
                {{getDuration(contest.start_time, contest.end_time)}}
              </li>
              <li>
                <Button size="small" shape="circle" @click="onRuleChange(contest.rule_type)">
                  {{contest.rule_type}}
                </Button>
              </li>
            </ul>
            </Col>
            <Col :span="4" style="text-align: center">
              <Tag type="dot" :color="CONTEST_STATUS_REVERSE[contest.status].color">{{$t('m.' + CONTEST_STATUS_REVERSE[contest.status].name.replace(/ /g, "_"))}}</Tag>
            </Col>
            </Row>
        </li>
      </ol>
    </Panel>
    <Pagination :total="total" :pageSize="limit" @on-change="getContestList" :current.sync="page"></Pagination>
    </Col>
  </Row>

</template>

<script>
  import api from '@oj/api'
  import { mapGetters } from 'vuex'
  import utils from '@/utils/utils'
  import Pagination from '@/pages/oj/components/Pagination'
  import time from '@/utils/time'
  import { CONTEST_STATUS_REVERSE, CONTEST_TYPE } from '@/utils/constants'

  const limit = 8

  export default {
    name: 'contest-list',
    components: {
      Pagination
    },
    data () {
      return {
        lectureID: '',
        page: 1,
        route_name: '',
        query: {
          status: '',
          keyword: '',
          rule_type: ''
        },
        limit: limit,
        total: 0,
        rows: '',
        contests: [],
        CONTEST_STATUS_REVERSE: CONTEST_STATUS_REVERSE,
//      for password modal use
        cur_contest_id: ''
      }
    },
    mounted () {
      this.lectureID = this.$route.params.lectureID
      this.route_name = this.$route.name
    },
    beforeRouteEnter (to, from, next) {
      api.getContestList(0, limit).then((res) => {
        next((vm) => {
          vm.contests = res.data.data.results
          vm.total = res.data.data.total
        })
      }, (res) => {
        next()
      })
    },
    methods: {
      init () {
        let route = this.$route.query
        this.query.status = route.status || ''
        this.query.rule_type = route.rule_type || ''
        this.query.keyword = route.keyword || 'test'
        this.page = parseInt(route.page) || 1
        this.getContestList()
      },
      getContestList (page = 1) {
        let offset = (page - 1) * this.limit
        api.getContestList(offset, this.limit, this.query).then((res) => {
          this.contests = res.data.data.results
          this.total = res.data.data.total
        })
      },
      changeRoute () {
        let query = Object.assign({}, this.query)
        query.page = this.page
        this.$router.push({
          name: 'contest-list',
          query: utils.filterEmptyValue(query)
        })
      },
      onRuleChange (rule) {
        this.query.rule_type = rule
        this.page = 1
        this.changeRoute()
      },
      onStatusChange (status) {
        this.query.status = status
        this.page = 1
        this.changeRoute()
      },
      goContest (contest) {
        this.cur_contest_id = contest.id
        if (contest.contest_type !== CONTEST_TYPE.PUBLIC && !this.isAuthenticated) {
          this.$error(this.$i18n.t('m.Please_login_first'))
          this.$store.dispatch('changeModalStatus', {visible: true})
        } else {
          this.$router.push({name: 'contest-details', params: {contestID: contest.id, lectureID: this.lectureID}})
        }
      },

      getDuration (startTime, endTime) {
        return time.duration(startTime, endTime)
      }
    },
    computed: {
      ...mapGetters(['isAuthenticated', 'user'])
    },
    watch: {
      '$route' (newVal, oldVal) {
        if (newVal !== oldVal) {
          this.init()
        }
      }
    }

  }
</script>
<style lang="less" scoped>
  #contest-card {
    #keyword {
      width: 80%;
      margin-right: 30px;
    }
    #no-contest {
      text-align: center;
      font-size: 16px;
      padding: 20px;
    }
    #contest-list {
      > li {
        padding: 20px;
        border-bottom: 1px solid rgba(187, 187, 187, 0.5);
        list-style: none;

        .trophy {
          height: 40px;
          margin-left: 10px;
          margin-right: -20px;
        }
        .contest-main {
          .title {
            font-size: 18px;
            a.entry {
              color: #495060;
              &:hover {
                color: #2d8cf0;
                border-bottom: 1px solid #2d8cf0;
              }
            }
          }
          li {
            display: inline-block;
            padding: 10px 0 0 10px;
            &:first-child {
              padding: 10px 0 0 0;
            }
          }
        }
      }
    }
  }
</style>