<template>
  <Row type="flex">
    <Col :span="24">
    <Panel id="lecture-card" shadow>
      <div slot="title"><b>{{$t('m.TextbookPractice')}}</b></div>
      <div slot="extra">
        <ul class="filter">
          <li>
            <Input id="keyword" @on-enter="changeRoute" @on-click="changeRoute" v-model="query.keyword"
                   icon="ios-search-strong" placeholder="Keyword"/>
          </li>
        </ul>
      </div>
      <ol id="lecture-list">
        <li><!--표시될 수강과목 수가 0이 아닌 경우에만 출력-->
          <Row id="tb-column" type="flex" justify="space-between" align="middle">
            <Col :span="12" style="padding-left: 150px">
              <p>교재명</p>
            </Col>
            <Col :span="2">
              <p>출판사</p>
			      </Col>
            <Col :span="4" style="text-align: center">
              교재실습 진행 버튼
			      </Col>
          </Row>
        </li>
        <li v-for="lecture in lectures" :key="lecture.lecture.id"><!--v-if 조건식을 통해 열림 상태인 수강 과목만 출력한다.-->
          <Row type="flex" justify="space-between" align="middle">
            <!--<img class="trophy" src="../../../../assets/Cup.png"/>--><!--트로피 대신 다른 이미지 추가-->
            <Col :span="12" class="lecture-main">
              <p class="title">
                <span class="entry" v-if="lecture.isallow" style="padding-left: 50px"><a id="lecture-title" @click="goLecture(lecture.lecture)">{{ lecture.lecture.title }}</a></span>
                <span id="waitlecture" class="entry" v-else>{{ lecture.lecture.title }}</span>
              </p>
            </Col>
            <Col :span="2">
              {{ lecture.lecture.created_by.realname }}
			      </Col>
            <Col :span="4" style="text-align: center">
              <Button @click="goLecture(lecture.lecture)" v-if="lecture.isallow">실습하기</Button>
              <Button v-else disabled>신청대기</Button>
			      </Col>
          </Row>
        </li>
      </ol>
      <p id="no-lecture" v-if="lectures.length == 0">{{$t('m.No_Book')}}</p>
    </Panel>
    <Pagination :total="total" :pageSize="limit" @on-change="getLectureList" :current.sync="page"></Pagination>
    </Col>
  </Row>

</template>

<script>
  import api from '@oj/api'
  import { mapGetters } from 'vuex'
  import utils from '@/utils/utils'
  import Pagination from '@/pages/oj/components/Pagination'
  import time from '@/utils/time'

  const limit = 8

  export default {
    name: 'lecture-list',
    components: {
      Pagination
    },
    data () {
      return {
        page: 1,
        yearsort: 2020,
        semestersort: 1,
        profsort: 0,
        query: {
          status: '',
          keyword: '',
          rule_type: ''
        },
        limit: limit,
        total: 0,
        rows: '',
        lectures: [],
//      for password modal use
        cur_lecture_id: '',
        sugaing: false,
        apptext: 'Apply'
      }
    },
    beforeRouteEnter (to, from, next) {
      let d = new Date()
      let semester = (((d.getMonth() + 1) <= 7 && (d.getMonth() + 1) >= 3) ? 1 : (((d.getMonth() + 1) <= 2 && (d.getMonth() + 1) >= 1) ? 3 : 2))
      api.getTakingLectureList(0, limit, undefined, d.getFullYear(), semester).then((res) => {
        next((vm) => {
          vm.lectures = res.data.data.results
          vm.total = res.data.data.total
        })
      }, (res) => {
        next()
      })
    },
    mounted () {
      let d = new Date()
      this.semestersort = (((d.getMonth() + 1) <= 7 && (d.getMonth() + 1) >= 3) ? 1 : (((d.getMonth() + 1) <= 2 && (d.getMonth() + 1) >= 1) ? 3 : 2))
      console.log(this.semestersort)
      this.yearsort = d.getFullYear()
    },
    methods: {
      init () {
        let route = this.$route.query
        this.query.rule_type = route.rule_type || ''
        this.query.keyword = route.keyword || ''
        this.page = parseInt(route.page) || 1
        this.getLectureList()
      },
      sortYear (year) {
        this.yearsort = year
        let route = this.$route.query
        this.query.rule_type = route.rule_type || ''
        this.query.keyword = route.keyword || ''
        this.page = 1
        this.getSortedLectureList(this.page)
      },
      sortSemester (semester) {
        this.semestersort = semester
        let route = this.$route.query
        this.query.rule_type = route.rule_type || ''
        this.query.keyword = route.keyword || ''
        this.page = 1
        this.getSortedLectureList(this.page)
      },
      getSortedLectureList (page = 1) {
        let offset = (page - 1) * this.limit
        api.getTakingLectureList(offset, this.limit, this.query, this.yearsort, this.semestersort, undefined).then((res) => {
          this.lectures = res.data.data.results
          this.total = res.data.data.total
        })
      },
      getLectureList (page = 1) {
        let offset = (page - 1) * this.limit
        api.getTakingLectureList(offset, this.limit, this.query, this.yearsort, this.semestersort, undefined).then((res) => {
          this.lectures = res.data.data.results
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
      onStatusChange (status) {
        this.query.status = status
        this.page = 1
        this.changeRoute()
      },
      goLecture (lecture) {
        this.cur_lecture_id = lecture.id
        this.$router.push({name: 'lecture-details', params: {lectureID: lecture.id, lectureTitle: lecture.title}})
      },
      applylecture (lecture) {
        if (!this.user.username) {
          this.$error('로그인 후 가능합니다.')
        } else {
          let data = {
            lecture_id: lecture.lecture.id,
            user_id: this.user.id,
            status: false
          }
          api.applyLecture(data).then(res => {
            this.getLectureList(this.page)
            this.$success('Success')
          })
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
  #listing {
    color: black
  }
  #waitlecture {
    color: #A4A4A4;
  }
  #lecture-card {
    #keyword {
      width: 80%;
      margin-right: 30px;
    }
    #no-lecture {
      text-align: center;
      font-size: 16px;
      padding: 20px;
    }
    #lecture-list {
      > li {
        padding: 20px;
        border-bottom: 1px solid rgba(187, 187, 187, 0.5);
        list-style: none;

        .trophy {
          height: 40px;
          margin-left: 10px;
          margin-right: -20px;
        }
        .lecture-main {
          .title {
            font-size: 18px;
            a.entry {
              color: #495060;
              &:hover {
                color: #407a2d;
                border-bottom: 1px solid #407a2d;
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
    #tb-column {
      font-weight: bold;
      font-size: 18px;
    }
    #lecture-title {
      font-weight: bold;
      color: black;
      &:hover {
        color: #407a2d;
      }
    }
  }

</style>
