<template>
  <div>
    <h1>Anggota Keluarga</h1>

    <div class="data-table mt-5">
      <v-card flat>
        <v-data-table
          :headers="headers"
          :items="familyMembersNotDeleted"
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
              <v-btn
                class="btn text-none ml-4 mt-2"
                color="blue accent-4"
                tag="router-link"
                to="tambah"
                dark
                depressed
              >
                Tambah anggota
              </v-btn>
            </v-card-title>
          </template>

          <!-- TABLE CONTENT -->
          <template v-slot:[`item.tglLahir`]="{ item }">
            {{ changeDate(item.tgl_lahir) }}
          </template>
          <template v-slot:[`item.telepon`]="{ item }">
            <p v-if="item === null">-</p>
            <p>{{ item }}</p>
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
                  <v-list-item @click="goToDetail(item.id)">
                    <v-list-item-title>Ubah</v-list-item-title>
                  </v-list-item>
                  <v-list-item @click="openConfirmDelete(item.id)">
                    <v-list-item-title>Hapus</v-list-item-title>
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

    <confirm-delete-modal
      @confirmDelete="confirmDeleteData"
    ></confirm-delete-modal>
  </div>
</template>

<script>
import { getData, deleteData, changeDateFormat } from '../../../utils'

export default {
  data: () => ({
    tableLoading: true,
    search: '',
    headers: [
      {
        text: 'Nama', value: 'nama',
      },
      {
        text: 'Tempat lahir', value: 'tempat_lahir',
      },
      {
        text: 'Tgl Lahir', value: 'tglLahir',
      },
      {
        text: 'Jenis kelamin', value: 'jenis_kelamin',
      },
      {
        text: 'Telepon', value: 'no_telp',
      },
      {
        text: 'Aksi', value: 'action',
      },
    ],
    familyMembers: [],
    page: 1,
    pageCount: 0,
    selectedJumlahData: 10,
    jumlahData: [10, 30, 50],
    deleteId: null,
  }),
  async mounted() {
    this.tableLoading = true

    this.familyMembers = await this.getAllFamilyMembers()
    
    this.tableLoading = false
  },
  computed: {
    familyMembersNotDeleted() {
      return this.familyMembers.filter(member => member.deleted_at === null)
    }
  },
  methods: {
    changeDate(date) {
      return changeDateFormat(date)
    },
    async getAllFamilyMembers() {
      try {
        let url = `/umat/keluarga/${this.$store.state.keluarga.id}`
        
        let response = await getData(url)

        return response
      } catch (e) {
        console.error(e)
      }
    },
    goToDetail(id) {
      this.$router.push(`/keluarga/anggota/${id}`)
    },
    openConfirmDelete(id) {
      this.deleteId = id
      this.$store.dispatch('deleteData/openModal')
    },
    async confirmDeleteData(decision) {
      // Close confirmation modal
      this.$store.commit('deleteData/resetModal')
      
      if (decision) {
        let snackbar = {}

        // Activate loading overlay
        this.$store.dispatch('loading/openLoading')

        try {
          let response = await deleteData('/umat', this.deleteId)
          
          if (response.status === 200) {
            snackbar.color = 'success'
            snackbar.text = 'Data berhasil dihapus'
            this.familyMembers = await this.getAllFamilyMembers()
          } else {
            snackbar.color = 'error'
            snackbar.text = 'Terjadi kesalahan. Silahkan refresh dan coba lagi'
          }
        } catch (error) {
          snackbar.color = 'error'
          snackbar.text = error
          console.error(error)
        }
        this.$store.dispatch('snackbar/openSnackbar', snackbar)
        this.$store.dispatch('loading/closeLoading')
      }
    }
  }
}
</script>