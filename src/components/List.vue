<template>
	<div class="list">
		<div class="loading" v-if="loading">Loading...</div>
		<div class="list-con">
			<div v-if="getdone" v-for="data in datas" style="font-size: 0px;" :key='data.id'>
				<div class="top-slide top" v-if="!!data.top_stories">
					<swiper :options="swiperOption" class="sdj">
						<swiper-slide class="top-img" v-for="slide in data.top_stories" :data-id="slide.id" :key='slide.id'>
							<div @click="toContent(slide.id)">
								<img :src="slide.image" :alt="slide.title">
								<div class="slide-content">
									<p>{{slide.title}}</p>
								</div>
							</div>
							
						</swiper-slide>
						<div class="swiper-pagination" slot="pagination"></div>
					</swiper>
				</div>
				<img class="other-img" v-if="!data.date && !!data.image" :src='data.image' :alt="data.name">
				<div class="list-message" v-if="!!data.stories">
					<p class="date-mes" v-if="!!data.date && !data.top_stories">{{data.date}}</p>
					<p class="data-editors" v-if="!!data.editors" @click="toEditor" :editors="data.editors">
						<span>主编</span>
						<ul>
							<li v-for="editor in data.editors" :key='editor.id'>
								<img :src="editor.avatar" alt="主编头像">
							</li>
						</ul>
					</p>
					<ul class="list-content">
						<li v-for="list in data.stories" @click="toContent(list.id)" :data-id="list.id" :key='list.id'>
							<p>{{list.title}}</p>
							<img v-if="!!list.images" :src="list.images" :alt="list.title">
						</li>
					</ul>
					
				</div>
			</div>
				

		</div>

		<div v-if="getdone" class="more-load" :class="{dbload: homepage}">loading...</div>
		
	</div>
</template>

<script>
	import api from "../api/index.js"
	import util from '../commons/util.js'
	import { mapState } from "vuex"

	export default {
		data() {
			return {
				topStories: [1, 2, 3, 4],
				datas: [],
	        	date: 0,
	        	routetimes: false,
	        	homepage: false,
	        	refresh: true,
				loading: false,
				getdone: false,
				swiperOption: {
					autoplay: 3500,
					setWrapperSize :true,
					pagination : '.swiper-pagination',
					paginationClickable :true,
					mousewheelControl : true,
					observeParents:true,
					loop: true,
				},
				slideNum: ":dd"
			}
		},
		created() {
			this.getlist();
		},
		mounted() {
			var rid = this.$route.path.split('\/')[2]
			this.routetimes = true
			if(!rid || rid === '0') {
				this.homepage = true
				this.$store.commit('setTitleName', '今日热闻')
				document.querySelector('.list').addEventListener('scroll',this.lazyLoad);
			} else {
				this.homepage = false
			}

			var _this = this;

			document.querySelector(".list").addEventListener("touchstart", util.tstart.bind(util))
			document.querySelector(".list").addEventListener("touchmove", util.tmove.bind(util))
			document.querySelector(".list").addEventListener("touchend", (function (e) {
				if(this.slideNum === 1) {
					_this.$store.commit('toggle', true);
				}
				this.slideNum = 0;
				this.num = 0;
			}).bind(util));
		},
		watch: {
			"$route": function(to, from) {
				if(/list/i.test(to.path)) {
					this.getlist()
				}
			}
		},
		methods: {
			toEditor() {
				this.$store.commit('setEditors', this.datas[0].editors)
				this.$router.push({
	        		name: "Editors",
	        	})
			},
			toContent(id) {
				if(id !== -1) {
					this.$router.push({
		        		name: "Content",
		        		params: {
		        			id: id
		        		}
		        	})
				}
			},
			getlist() {
				var rid = this.$route.path.split('\/')[2]
				if(this.routetimes) {
					if(!rid || rid === '0') {
						this.homepage = true
						document.querySelector('.list').addEventListener('scroll',this.lazyLoad);
					} else if(this.homepage) {
						this.homepage = false
						document.querySelector('.list').removeEventListener('scroll',this.lazyLoad);
					}
				}

				this.datas = [];
				this.date = 0;
				var that = this;
				var jumpId = this.$route.params.id
				var local = "newsThemeDetail"
				this.loading = true
				this.getdone = false

				if(!jumpId || jumpId === '0') {
					jumpId = ""
					local = "news"
				}

				if(!/list/i.test(this.$route.path)) {
					return
				}

				api.getMessage(local, jumpId).then(function(data) {
					var data = data.data;
					that.$store.commit('setTitleName', data.name )
					if(!!data.top_stories) {
						var top_stories = data.top_stories;
						var tlength = top_stories.length;
						data.top_stories = top_stories
					}
					if(!!data.stories) {
						var stories = data.stories;
						var tlength = stories.length;
						data.stories = stories
					}

					if(!!data.date) {
						var date = data.date
						date = date.substring(0, 4) + "年" + date.substring(4, 6) + "月" + date.substring(6, 8) + "日"
						data.date = date
					}
						
					that.datas.push(data)
					that.loading = false
					that.getdone = true
					
					
				}).catch(err => {
					console.log(err);
				});
			},
			lazyLoad(e) {
				var that = this,
					clientH = document.documentElement.clientHeight || document.body.clientHeight,
					scrollH = document.querySelector('.list').scrollTop,
					eleTop = document.querySelector('.more-load').offsetTop,
					arr = [];

				//标题Title信息更新
				document.querySelectorAll('.date-mes').forEach(function(v,i){
					if(v.getBoundingClientRect().top < 10) {
						arr.push(v);
					}
				});

				!!arr[0] ? this.$store.commit('setTitleName', arr[arr.length - 1].innerHTML) : this.$store.commit('setTitleName', '今日热闻')

				if(eleTop - 100 <= (clientH + scrollH) && that.refresh) {
					that.refresh = false;
					var gd = that.getdate();
					api.getMessage('newsDate', gd).then((data) => {
						that.datas.push(data.data)
						that.date += 1
						that.refresh = true
					}).catch(err => {
						console.log(':', err);
					})
				}
			},
			getdate() {
				var dt = new Date();
				dt.setDate(dt.getDate() - this.date);
				var y = dt.getFullYear();
				var m = dt.getMonth() + 1;
				m = m >= 10 ? m : '0' + m;
				var d = dt.getDate();
				d = d >= 10 ? d : '0' + d;
				return '' + y + m + d;
			}

		}
	}

		
</script>

<style lang="scss">
	p {
		margin: 0;
	}
	.list {
		position: absolute;
		top: 0;
		width: 100%;
		height: 100%;
		z-index: 1;
		overflow-y: auto;
		-webkit-overflow-scrolling: touch;

		.loading {
			position: absolute;
			width: 100%;
			font-size: 34px;
			padding-top: 400px;
		}

		.other-img {
			
		}

		.list-message {
			font-size: 0;

			.data-editors {
				display: flex;
				flex-flow: row nowrap;
				align-items: center;
				line-height: 60px;
				height: 60px;
				font-size: 25px;
				text-align: left;
				padding: 0 43px 0 34px;

				&:after {
					content: "";
					flex: 0 0 15px;
		        	height: 15px;
		        	border: {
			        	top: 2px solid #6f6f6f;
			        	right: 2px solid #6f6f6f;
			        }
			        transform: rotate(45deg);
				}

				span {
					font-size: 24px;
					color: #6f6f6f;
					flex: 0 0 60px;
				}

				ul {
					flex: 1;
					margin: 0 30px;
					padding: 0;

					li {
						display: inline-block;

						img {
							vertical-align: middle;
							height: 34px;
							margin: 8px 10px 12px;
							border-radius: 33px;
						}
					}
				}
			}

			.date-mes {
				background-color: #5886e8;
				height: 58px;
				line-height: 58px;
				font-size: 28px;
				color: #fff;
				font-weight: bold;
			}
			ul.list-content {
				margin: 0 30px;
				padding: 0;

				li {
					display: flex;
					align-items: center;
					height: 136px;
					padding: 3px 0;
					-webkit-tap-highlight-color:rgba(0,0,0,0);
					border: {
						top: 1px solid #ddd;
					} 

					p {
						flex: 1;
						text-align: justify;
						line-height: 24px;
						font-size: 23px;
						padding: 0 14px 0;
						font-weight: bold;
					}

					img {
						width: 111px;
						overflow-y: hidden;
						align-items: center;
						vertical-align: middle;
					}
				}
			}

		}
	}
	.more-load {
		display: none;
		line-height: 70px;
		font-size: 23px;
		text-align: center;
		font-weight: bold;
		height: 70px;
		padding: 3px 0;
		border: {
			top: 1px solid #ddd;
		}
	}
	.dbload {
		display: block;
	}
	.top {
		position: relative;
		height: 350px;
		.top-img {
			position: relative;
			width: 100%;
			height: 350px;
			color: #000;
			background-color: #ddd;
			line-height: 440px;
			overflow: hidden;
			
			img {
				width: 100%;
				margin-top: -145px;
			}
		}
		.sdj {
			position: relative;
			width: 100%;
			height: 100%;
		}
		.slide-content {
			position: absolute;
			width: 100%;
			height: 100%;
			top: 0;
			background-color: rgba(0, 0, 0, .3);

			p {
				position: absolute;
				font-size: 29px;
				color: #fff;
				bottom: 23px;
				text-align: left;
				line-height: 44px;
				max-width: 600px;
				left: 50%;
				margin-left: -300px;
			}
		}

	}
		
</style>