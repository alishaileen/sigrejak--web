<template>
  <div>
    <btn-kembali path="/pengurus/surat" />
      
    <h1>Surat Komuni I/Penguatan</h1>

    <div class="data-table mt-5">
      <v-card flat>
        <v-data-table
          :headers="headers"
          :items="suratList"
          :search="search"
          :page.sync="page"
          :items-per-page="selectedJumlahData"
          :loading="tableLoading"
          loading-text="Memuat data ..."
          hide-default-footer
          @page-count="pageCount = $event"
        >
          <!-- TABLE TOP -->
          <template v-slot:top>
            <v-card-title>
              <v-row dense>
                <v-col>
                  <v-text-field
                    v-model="search"
                    prepend-inner-icon="mdi-magnify"
                    label="Cari"
                    single-line
                    hide-details
                    outlined
                    dense
                    background-color="#FAFAFA"
                  ></v-text-field>
                </v-col>
                <v-col cols="4">
                  <v-combobox
                    v-model="selectedLingkungan"
                    :items="lingkunganList"
                    item-text="nama_lingkungan"
                    label="Filter lingkungan"
                    color="blue"
                    outlined 
                    dense
                    hide-details
                    single-line
                    multiple
                    small-chips
                    clearable
                    deletable-chips
                  ></v-combobox>
                </v-col>
                <v-col cols="3">
                  <v-select
                    v-model="selectedShowData"
                    :items="[ 'Semua', 'Belum diverifikasi', 'Terverifikasi' ]"
                    outlined
                    dense
                    hide-details
                    label="Tampil data"
                  ></v-select>
                </v-col>
              </v-row>
            </v-card-title>
          </template>

          <!-- TABLE CONTENT -->
          <template v-slot:[`item.jadwal_baptis`]="{ item }">
              <v-chip
                v-if="item.tgl_baptis != null"
                color="green"
              >
                {{ item.tgl_baptis }}
              </v-chip>
              <v-chip v-else>
                Belum ada
              </v-chip>
          </template>
          <template v-slot:[`item.jenis`]="{ item }">
            <v-chip
              v-if="item.jenis_surat === 1"
              color="lime"
            >
              Komuni I
            </v-chip>
            <v-chip
              v-else
              color="orange lighten-3"
            >
              Penguatan
            </v-chip>
          </template>
          <template v-slot:[`item.status_sekretariat`]="{ item }">
            <span v-if="item.sekretariat_approval === 1" class="d-flex justify-center">
              <v-icon color="green darken-2">far fa-check-circle</v-icon>
            </span>
            <span v-else>
              <v-icon color="grey darken-2">fas fa-history</v-icon>
            </span>
          </template>
          <template v-slot:[`item.action`]="{ item }">
            <div>
              <v-menu bottom offset-y>
                <template v-slot:activator="{ on, attrs }">
                  <v-btn text fab small v-bind="attrs" v-on="on">
                    <v-icon color="#131313">mdi-dots-vertical</v-icon>
                  </v-btn>
                </template>
                <v-list>
                  <v-list-item @click="openModalDetail(item)">
                    <v-list-item-title>Detail</v-list-item-title>
                  </v-list-item>
                  <v-list-item
                    v-if="item.ketua_lingkungan_approval === 1 && item.sekretariat_approval === 1"
                    @click="cetak(item.id)"
                  >
                    <v-list-item-title>Cetak surat</v-list-item-title>
                  </v-list-item>
                </v-list>
              </v-menu>
            </div>
          </template>

          <template v-slot:footer>
            <v-divider></v-divider>
            <div class="d-flex justify-start text-center py-2">
              <v-pagination
                class="table-pagination ml-2 mb-3"
                v-model="page"
                :length="pageCount"
                :total-visible="6"
                color="blue accent-4"
              ></v-pagination>
            </div>
          </template>
        </v-data-table>
      </v-card>
    </div>

    <snackbar></snackbar>

    <modal-detail
      :isModalDetailActive="isModalDetailActive"
      :data="selectedDetail"
      :url="url"
      :sekretariat="sekretariat"
      :romoPembaptis="romoPembaptis"
      @verify="sekretariatVerify"
      @openImage="openCloseModal"
      @closeModal="closeModalDetail"
    ></modal-detail>

    <dialog-image
      :isDialogActive="isImageCardActive"
      :imageSrc="displayGambarAkta"
      @close="openCloseModal"
    ></dialog-image>
  </div>
</template>

<script>
import { API_URL } from '../../../../constants'
import { getData, getOneData, cetakSurat } from '../../../../utils'
import { verifySurat } from '../../../../utils/pengurus'

import ModalDetail from './DetailModal'
import DialogImage from '../../../../components/DialogImage.vue'

export default {
  components: {
    ModalDetail,
    DialogImage,
  },
  data: () => ({
    url: '/surat-komuni-penguatan',
    selectedLingkungan: [],
    selectedShowData: 'Semua',
    tableLoading: true,
    search: '',
    headers: [
      {
        text: 'Tgl surat', value: 'created_at',
      },
      {
        text: 'Nama', value: 'nama', width: '200'
      },
      {
        text: 'Ayah', value: 'nama_ayah', width: '100'
      },
      {
        text: 'Ibu', value: 'nama_ibu', width: '100'
      },
      {
        text: '', value: 'jenis',
      },
      {
        text: 'Sekretariat', value: 'status_sekretariat', align: 'center', sortable: false
      },
      {
        text: 'Aksi', value: 'action', sortable: false
      },
    ],
    surat: [],
    page: 1,
    pageCount: 0,
    selectedJumlahData: 10,
    jumlahData: [10, 30, 50],
    deleteId: null,

    isModalDetailActive: false,
    selectedDetail: {},
    sekretariat: {},
    romoPembaptis: {},
    lingkunganList: [],

    displayGambarAkta: null,
    isImageCardActive: false,
  }),
  computed: {
    suratList() {
      if (this.selectedShowData === 'Semua') {
        return this.surat.filter(e => {
          if(this.selectedLingkungan.length === 0) {
            return e.deleted_at === null && e.ketua_lingkungan_approval === 1
          } else {
            return e.deleted_at === null && e.ketua_lingkungan_approval === 1 && this.selectedLingkungan.some(el => el.id === e.id_lingkungan)
          }
        })
      } else if (this.selectedShowData === 'Belum diverifikasi') {
        return this.surat.filter(e => {
          if(this.selectedLingkungan.length === 0) {
            return e.deleted_at === null && e.sekretariat_approval === null
          } else {
            return e.deleted_at === null && e.sekretariat_approval === null && this.selectedLingkungan.some(el => el.id === e.id_lingkungan)
          }
        })
      } else {
        return this.surat.filter(e => {
          if(this.selectedLingkungan.length === 0) {
            return e.deleted_at === null && e.sekretariat_approval === 1
          } else {
            return e.deleted_at === null && e.sekretariat_approval === 1 && this.selectedLingkungan.some(el => el.id === e.id_lingkungan)
          }
        })
      }
    }
  },
  async mounted() {
    this.tableLoading = true
    this.surat = await getData(this.url)
    this.lingkunganList = await getData(`/lingkungan`)
    this.tableLoading = false
  },
  methods: {
    openCloseModal(bool, fileName = null) {
      if(bool === true) {
        this.displayGambarAkta = `${API_URL}/files/${fileName}`
      }
      this.isImageCardActive = bool
    },

    async openModalDetail(data) {
      this.selectedDetail = data

      if(data.id_sekretariat != null) {
        this.sekretariat = await getOneData(`/admin/${data.id_sekretariat}`)
      }
      if(data.id_romo_pembaptis != null) {
        this.romoPembaptis = await getOneData(`/admin/${data.id_romo_pembaptis}`)
      }

      this.isModalDetailActive = true
    },
    async closeModalDetail() {
      this.isModalDetailActive = false
    },
    async sekretariatVerify(dataSurat) {
      this.$store.dispatch('loading/openLoading')
      this.$store.commit('snackbar/resetSnackbar')

      let snackbar = {}
      
      dataSurat.role = 'sekretariat'
      dataSurat.id_sekretariat = this.$store.state.pengurus.id
      snackbar = await verifySurat(this.url, dataSurat.id, dataSurat)
      
      this.surat = await getData(this.url)
      
      this.closeModalDetail()
      
      this.$store.dispatch('snackbar/openSnackbar', snackbar)
      this.$store.dispatch('loading/closeLoading')
    },
    async cetak(id) {
      this.$store.dispatch('loading/openLoading')
      
      let link = await cetakSurat(this.url, id)
      
      this.$store.dispatch('loading/closeLoading')

      window.open(link)
    },
  }
}
</script>