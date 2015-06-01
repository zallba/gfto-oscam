# definire  MODULE_LOG_PREFIX  " dvbapi "
# includono  " globals.h "
# ifdef HAVE_DVBAPI
# includono  " modulo-dvbapi.h "
# includere  " modulo-cacheex.h "
# includono  " modulo-dvbapi-azbox.h "
# includono  " modulo-dvbapi-mca.h "
# includono  " modulo-dvbapi-coolapi.h "
# includono  " modulo-dvbapi-stapi.h "
# includono  " modulo-dvbapi-chancache.h "
# includono  " modulo-stat.h "
# includono  " oscam-chk.h "
# includono  " oscam-client.h "
# includono  " oscam-config.h "
# includono  " oscam-ecm.h "
# includono  " oscam-emm.h "
# includono  " oscam-files.h "
# includono  " oscam-net.h "
# includono  " oscam-reader.h "
# includono  " oscam-string.h "
# includono  " oscam-time.h "
# includono  " oscam-work.h "
# includono  " lettore-irdeto.h "

# se definito (__CYGWIN__)
# definire  F_NOTIFY  0
# definire  F_SETSIG  0
# definire  DN_MODIFY  0
# definire  DN_CREATE  0
# definire  DN_DELETE  0
# definire  DN_MULTISHOT  0
# endif

static  int is_samygo;

static  int  dvbapi_ioctl ( int fd, uint32_t richiesta, ...)
{ 
	int ret = 0 ;
	va_list args;
	va_start (args, richiesta);
	se (! is_samygo)
	{
		vuoto * param = va_arg (args, vuoto *);
		ret = ioctl (fd, richiesta, param);
	} 
	altro 
	{
		interruttore (richiesta)
		{
			caso DMX_SET_FILTER:
			{
				struct dmxSctFilterParams * SFP = va_arg (args, struct dmxSctFilterParams *);
				// Preparare pacchetto
				unsigned  char pacchetto [ sizeof (richiesta) + sizeof ( struct dmxSctFilterParams)];
				memcpy (e del pacchetto, e richiesta, sizeof (richiesta));
				memcpy (e packet [ sizeof (richiesta)], sfp , sizeof ( struct dmxSctFilterParams));
				ret = invio (fd, pacchetto, sizeof (pacchetto), 0 );
				rompere ;
			}
			caso DMX_SET_FILTER1:
			{
				struct dmx_sct_filter_params * SFP = va_arg (args, struct dmx_sct_filter_params *);
				ret = invio (fd, SFP , sizeof ( struct dmx_sct_filter_params), 0 );
				rompere ;
			}
			caso DMX_STOP:
			{
				ret = invio (fd, e richiesta, sizeof (richiesta), 0 );
				ret = 1 ;
				rompere ;
			}
			caso CA_SET_PID:
			{
				ret = 1 ;
				rompere ;
			}
			caso CA_SET_DESCR:
			{
				ret = 1 ;
				rompere ;
			}
		}
		se (ret> 0 ) // send () può restituire maggiore di 1
			ret = 1 ;
	}
# se definito (__ powerpc__)
	// vecchie scatole DM500 (ppc vecchio) utilizzano kernel rotto, se abbiamo bisogno di alcune correzioni
	interruttore (richiesta)
	{
		caso DMX_STOP:
		caso CA_SET_DESCR:
		caso CA_SET_PID:
		ret = 1 ;
	}
# endif
	// FIXME: Soluzione per su980 bug
	// Vedi: http://www.streamboard.tv/wbb2/thread.php?postid=533940
	se ( boxtype_is ( " su980 " ))
		ret = 1 ;
	va_end (args);
	ritorno ret;
}

// Tunemm_caid_map
# definire  from_to  0
# definire  TO_FROM  1

int32_t pausecam = 0 , disable_pmt_files = 0 , pmt_stopmarking = 0 , pmthandling = 0 ;
DEMUXTYPE demux [MAX_DEMUX];
struct s_dvbapi_priority * dvbapi_priority;
struct s_client * dvbapi_client;

const  char * boxdesc [] = { " none " , " dreambox " , " duckbox " , " ufs910 " , " dbox2 " , " ipbox " , " ipbox-pmt " , " dm7000 " , " qboxhd " , " coolstream " , " neumo " , " pc " , " pc-nodmx " };

static  const  struct dispositivi box_devices [BOX_COUNT] =
{
	/ * QboxHD (dvb-api-3) * /      { " / tmp / virtual_adapter / " ,   " ca % d " ,          " demux % d " ,       " /tmp/camd.socket " , DVBAPI_3},
	/ * Dreambox (dvb-api-3) * /    { " / dev / dvb / adattatore % d / " ,     " ca % d " ,          " demux % d " ,       " /tmp/camd.socket " , DVBAPI_3},
	/ * Dreambox (dvb-api-1) * /    { " / dev / dvb / carta % d / " ,        " ca % d " ,          " demux % d " ,       " /tmp/camd.socket " , DVBAPI_1},
	/ * NEUMO (dvb-api-1) * /       { " / dev / " ,                   " demuxapi " ,      " demuxapi " ,      " /tmp/camd.socket " , DVBAPI_1},
	/ * SH4 (STAPI) * /        { " / dev / Stapi / " ,             " stpti4_ioctl " , " stpti4_ioctl " , " /tmp/camd.socket " , STAPI},
	/ * * CoolStream /              { " / dev / CNXT / " ,              " nullo " ,          " nullo " ,          " /tmp/camd.socket " , COOLAPI}
};

static  int32_t selected_box = - 1 ;
static  int32_t selected_api = - 1 ;
static  int32_t maxfilter = MAX_FILTER;
static  int32_t dir_fd = - 1 ;
char * client_name = NULL ;
static  uint16_t client_proto_version = 0 ;

static  int32_t ca_fd [MAX_DEMUX]; // detiene manico fd di ogni dispositivo ca 0 = non è in uso
statica LLIST * ll_activestreampids; // elenco di tutti streampids abilitati sui dispositivi ca

static  int32_t unassoc_fd [MAX_DEMUX];

bool  is_dvbapi_usr ( char * usr) {
	ritorno  streq (. cfg dvbapi_usr , usr);
}

struct s_emm_filter
{
	int32_t       demux_id;
	uchar filtrare [ 32 ];
	uint16_t      caid;
	uint32_t      provid;
	uint16_t      pid;
	uint32_t      num;
	struct voltaB time_started;
};

statica LLIST * ll_emm_active_filter;
statica LLIST * ll_emm_inactive_filter;
statica LLIST * ll_emm_pending_filter;

int32_t  add_emmfilter_to_list ( int32_t demux_id, uchar * filtro, uint16_t caid, uint32_t provid, uint16_t emmpid, int32_t num, bool enable)
{
	se (ll_emm_active_filter!)
		{Ll_emm_active_filter = ll_create ( " ll_emm_active_filter " ); }

	se (ll_emm_inactive_filter!)
		{Ll_emm_inactive_filter = ll_create ( " ll_emm_inactive_filter " ); }

	se (ll_emm_pending_filter!)
		{Ll_emm_pending_filter = ll_create ( " ll_emm_pending_filter " ); }

	struct s_emm_filter * filter_item;
	se (! cs_malloc (& filter_item, sizeof ( struct s_emm_filter)))
		{ ritorno  0 ; }

	filter_item-> demux_id = demux_id;
	memcpy (filter_item->, filtro, 32 );
	filter_item-> caid = caid;
	filter_item-> provid = provid;
	filter_item-> pid = emmpid;
	filter_item-> num = num;
	se (attiva)
	{
		cs_ftime (e filter_item-> time_started);
	}
	altro
	{
		memset (& filter_item-> time_started, 0 , sizeof (filter_item-> time_started));
	}
		
	se (num> 0 )
	{
		ll_append (ll_emm_active_filter, filter_item);
		cs_log_dbg (D_DVBAPI, " Demuxer % d Filtra % d aggiunto emmfilters attivi (CAID % 04X PROVID % 06X EMMPID % 04X ) " ,
					  filter_item-> demux_id, filter_item-> num, filter_item-> caid, filter_item-> provid, filter_item-> pid);
	}
	altrimenti  se (num < 0 )
	{
		ll_append (ll_emm_pending_filter, filter_item);
		cs_log_dbg (D_DVBAPI, " Demuxer % d Filter aggiunto emmfilters pendenti (CAID % 04X PROVID % 06X EMMPID % 04X ) " ,
					  filter_item-> demux_id, filter_item-> caid, filter_item-> provid, filter_item-> pid);
	}
	altro
	{
		ll_append (ll_emm_inactive_filter, filter_item);
		cs_log_dbg (D_DVBAPI, " Demuxer % d Filter aggiunto emmfilters inattivi (CAID % 04X PROVID % 06X EMMPID % 04X ) " ,
					  filter_item-> demux_id, filter_item-> caid, filter_item-> provid, filter_item-> pid);
	}
	ritorno  1 ;
}

int32_t  is_emmfilter_in_list_internal (LLIST * ll, uchar * filtro, uint16_t emmpid, uint32_t provid, uint16_t caid)
{
	struct s_emm_filter * filter_item;
	LL_ITER itr;
	se ( ll_count (ll)> 0 )
	{
		itr = ll_iter_create (II);
		mentre ((filter_item = ll_iter_next (& ITR))! = NULL )
		{
			se (! memcmp (filter_item->, filtro, 32 ) && (filter_item-> pid == emmpid) && (filter_item-> provid == provid) && (filter_item-> caid == Caid))
			{ ritorno  1 ; }
		}
	}
	ritorno  0 ;
}

int32_t  is_emmfilter_in_list (uchar * filtro, uint16_t emmpid, uint32_t provid, uint16_t caid)
{
	se (ll_emm_active_filter!)
		{Ll_emm_active_filter = ll_create ( " ll_emm_active_filter " ); }

	se (ll_emm_inactive_filter!)
		{Ll_emm_inactive_filter = ll_create ( " ll_emm_inactive_filter " ); }

	se (ll_emm_pending_filter!)
		{Ll_emm_pending_filter = ll_create ( " ll_emm_pending_filter " ); }

	se ( is_emmfilter_in_list_internal (ll_emm_active_filter, filtro, emmpid, provid, Caid))
		{ ritorno  1 ; }
	se ( is_emmfilter_in_list_internal (ll_emm_inactive_filter, filtro, emmpid, provid, Caid))
		{ ritorno  1 ; }
	se ( is_emmfilter_in_list_internal (ll_emm_pending_filter, filtro, emmpid, provid, Caid))
		{ ritorno  1 ; }

	ritorno  0 ;
}

struct s_emm_filter * get_emmfilter_by_filternum_internal (LLIST * ll, int32_t demux_id, uint32_t num)
{
	struct s_emm_filter * filtro;
	LL_ITER itr;
	se ( ll_count (ll)> 0 )
	{
		itr = ll_iter_create (II);
		mentre ((filtro = ll_iter_next (& ITR)))
		{
			se (Filtro-> demux_id == demux_id && Filtro-> num == num)
				{ ritorno del filtro; }
		}
	}
	ritorno  NULL ;
}

struct s_emm_filter * get_emmfilter_by_filternum ( int32_t demux_id, uint32_t num)
{
	se (ll_emm_active_filter!)
		{Ll_emm_active_filter = ll_create ( " ll_emm_active_filter " ); }

	se (ll_emm_inactive_filter!)
		{Ll_emm_inactive_filter = ll_create ( " ll_emm_inactive_filter " ); }

	se (ll_emm_pending_filter!)
		{Ll_emm_pending_filter = ll_create ( " ll_emm_pending_filter " ); }

	struct s_emm_filter * emm_filter = NULL ;
	emm_filter = get_emmfilter_by_filternum_internal (ll_emm_active_filter, demux_id, num);
	se (emm_filter)
		{ ritorno emm_filter; }
	emm_filter = get_emmfilter_by_filternum_internal (ll_emm_inactive_filter, demux_id, num);
	se (emm_filter)
		{ ritorno emm_filter; }
	emm_filter = get_emmfilter_by_filternum_internal (ll_emm_pending_filter, demux_id, num);
	se (emm_filter)
		{ ritorno emm_filter; }

	ritorno  NULL ;
}

int8_t  remove_emmfilter_from_list_internal (LLIST * ll, int32_t demux_id, uint16_t caid, uint32_t provid, uint16_t pid, uint32_t num)
{
	struct s_emm_filter * filtro;
	LL_ITER itr;
	se ( ll_count (ll)> 0 )
	{
		itr = ll_iter_create (II);
		mentre ((filtro = ll_iter_next (& ITR)))
		{
			se (Filtro-> demux_id == demux_id && Filtro-> caid == caid && Filtro-> provid == provid && Filtro-> pid == pid && Filtro-> num == num)
			{
				ll_iter_remove_data (& ITR);
				ritorno  1 ;
			}
		}
	}
	ritorno  0 ;
}

vuoto  remove_emmfilter_from_list ( int32_t demux_id, uint16_t caid, uint32_t provid, uint16_t pid, uint32_t num)
{
	se (ll_emm_active_filter && remove_emmfilter_from_list_internal (ll_emm_active_filter, demux_id, caid, provid, pid, num))
		{ ritorno ; }
	se (ll_emm_inactive_filter && remove_emmfilter_from_list_internal (ll_emm_inactive_filter, demux_id, caid, provid, pid, num))
		{ ritorno ; }
	se (ll_emm_pending_filter && remove_emmfilter_from_list_internal (ll_emm_pending_filter, demux_id, caid, provid, pid, num))
		{ ritorno ; }
}

int32_t  dvbapi_net_send ( uint32_t request, int32_t socket_fd, int32_t demux_index, uint32_t filter_number, unsigned  char *data)
{
	unsigned  char pacchetto [ 262 ];                                           // massima dimensione possibile del pacchetto
	int32_t size = 0 ;

	// Non collegato?
	se (socket_fd <= 0 )
		ritorno  0 ;

	// Preparazione del pacchetto - intestazione
	// Nel vecchio client protocollo aspettiamo che questo primo byte come indice adattatore, cambiato nel nuovo protocollo
	// Per essere sempre dopo la richiesta tipo (codice operativo)
	se (client_proto_version <= 0 )
		. packet [size ++] = demux [demux_index] adapter_index ;           // indice adattatore - 1 byte

	// Tipo di richiesta
	uint32_t req = richiesta;
	se (client_proto_version> = 1 )
		req = htonl (req);
	memcpy (e del pacchetto [size], e req, 4 );                                      // richiesta - 4 byte
	size = + 4 ;

	// Preparazione del pacchetto - Indice di adattatore per il proto> = 1
	se ((richiesta! = DVBAPI_SERVER_INFO) && client_proto_version> = 1 )
		. packet [size ++] = demux [demux_index] adapter_index ;           // indice adattatore - 1 byte

	// Struct con i dati
	interruttore (richiesta)
	{
		caso DVBAPI_SERVER_INFO:
		{
			int16_t proto_version = htons (DVBAPI_PROTOCOL_VERSION);            // nostra versione del protocollo
			memcpy (e del pacchetto [size], e proto_version, 2 );
			size = + 2 ;

			unsigned  char * info_len = & pacchetto [size];    // Info lunghezza della stringa
			size = + 1 ;

			* Info_len = snprintf (( char *) e packet [size], sizeof (pacchetto) - dimensioni, " OSCam v % s , costruire r % s ( % s ) " , CS_VERSION, CS_SVN_VERSION, CS_TARGET);
			size = + * info_len;
			rompere ;
		}
		caso DVBAPI_CA_SET_PID:
		{
			int sct_capid_size = sizeof ( ca_pid_t );

			se (client_proto_version> = 1 )
			{
				ca_pid_t * capid = ( ca_pid_t *) dati;
				capid-> pid = htonl (capid-> pid);
				capid-> index = htonl (capid-> index );
			}
			memcpy (e del pacchetto [size], dati, sct_capid_size);

			size = + sct_capid_size;
			rompere ;
		}
		caso DVBAPI_CA_SET_DESCR:
		{
			int sct_cadescr_size = sizeof ( ca_descr_t );

			se (client_proto_version> = 1 )
			{
				ca_descr_t * cadesc = ( ca_descr_t *) dati;
				cadesc-> index = htonl (cadesc-> index );
				cadesc-> parità = htonl (cadesc-> parità);
			}
			memcpy (e del pacchetto [size], dati, sct_cadescr_size);

			size = + sct_cadescr_size;
			rompere ;
		}
		caso DVBAPI_DMX_SET_FILTER:
		caso DVBAPI_DMX_STOP:
		{
			int32_t sct_filter_size = sizeof ( struct dmx_sct_filter_params);
			pacchetto [size ++] = demux_index;                                // demux indice - 1 byte
			pacchetto [size ++] = filter_number;                              // numero di filtro - 1 byte

			se (i dati)        // dati filtro all'avvio
			{
				se (client_proto_version> = 1 )
				{
					struct dmx_sct_filter_params * fp = ( struct dmx_sct_filter_params *) dati;

					// Aggiungendo tutti i campi della struttura dmx_sct_filter_params
					// Uno ad uno per evitare problemi padding
					uint16_t pid = htons (FP> pid);
					memcpy (e del pacchetto [size], e pid, 2 );
					size = + 2 ;

					memcpy (e del pacchetto [size], FP> Filtro. Filtro , 16 );
					size = + 16 ;
					memcpy (e del pacchetto [size], FP> Filtro. maschera , 16 );
					size = + 16 ;
					memcpy (. & pacchetto, FP> filtro [size] mode , 16 );
					size = + 16 ;

					uint32_t timeout = htonl (FP> timeout);
					memcpy (e del pacchetto [size], e timeout, 4 );
					size = + 4 ;

					uint32_t flags = htonl (FP> bandiere);
					memcpy (e del pacchetto [size], e le bandiere, 4 );
					size = + 4 ;
				}
				altro
				{
					memcpy (e del pacchetto [size], dati, sct_filter_size);        // dmx_sct_filter_params struct
					size = + sct_filter_size;
				}
			}
			altro             // pid quando si arresta
			{
				se (client_proto_version> = 1 )
				{
					uint16_t pid = htons (. demux [demux_index] demux_fd [filter_number]. pid );
					memcpy (e del pacchetto [size], e pid, 2 );
					size = + 2 ;
				}
				altro
				{
					uint16_t . pid = demux [demux_index] demux_fd [filter_number]. pid ;
					pacchetto [size ++] = pid >> 8 ;
					pacchetto [size ++] = pid & 0xFF;
				}
			}
			rompere ;
		}
		predefinito :   // richiesta sconosciuta
		{
			cs_log ( " ERRORE: dvbapi_net_send: richiesta non valida " );
			ritorno  0 ;
		}
	}

	// Invio
	cs_log_dump_dbg (D_DVBAPI, pacchetto, le dimensioni, " Invio pacchetto dvbapi client (fd = % d ): " , socket_fd);
	inviare (socket_fd, e dei pacchetti, formato, MSG_DONTWAIT);

	// Ritorno sempre successo come il client potrebbe chiudere presa
	ritorno  0 ;
}

int32_t  dvbapi_set_filter ( int32_t demux_id, int32_t api, uint16_t pid, uint16_t caid, uint32_t provid, uchar * filt, uchar * maschera, int32_t timeout, int32_t pidindex, int32_t tipo,
	int8_t add_to_emm_list)
{
	openxcas_set_caid (demux. [demux_id] ECMpids [pidindex]. CAID );
	openxcas_set_ecm_pid (pid);
	se (USE_OPENXCAS)
		ritorno  1 ;

	int32_t ret = - 1 , n = - 1 , i;

	per (i = 0 ; i <maxfilter && demux [demux_id]. demux_fd [i]. fd > 0 ; i ++) {; }

	se (i> = maxfilter)
	{
		cs_log_dbg (D_DVBAPI, " nessun filtro libero " );
		ritorno - 1 ;
	}
	n = i;

	. demux [demux_id] demux_fd [n]. pidindex = pidindex;
	. demux [demux_id] demux_fd [n]. pid       = pid;
	. demux [demux_id] demux_fd [n]. caid      = caid;
	. demux [demux_id] demux_fd . [n] provid    = provid;
	. demux [demux_id] demux_fd [n]. Tipo      = tipo;

	interruttore (api)
	{
	caso DVBAPI_3:
		se (cfg. dvbapi_listenport || cfg. dvbapi_boxtype == BOXTYPE_PC_NODMX)
			ret = demux [demux_id]. demux_fd [n]. fd = DUMMY_FD;
		altro
			ret = demux [demux_id]. demux_fd [n]. fd = dvbapi_open_device ( 0 , demux [demux_id]. demux_index , demux [demux_id]. adapter_index );
		se (ret < 0 ) { ritorno ret; }   // restituisce se dispositivo cant essere aperta!
		struct dmx_sct_filter_params SFP2 ;

		memset (& SFP2 , 0 , sizeof ( SFP2 ));

		SFP2 . pid             = pid;
		SFP2 . Timeout         = timeout;
		SFP2 . flags           = DMX_IMMEDIATE_START;
		se (cfg. dvbapi_boxtype == BOXTYPE_NEUMO)
		{
			// DeepThought: sulle immagini DGS / CubeStation e NEUMO, forse altri
			// Il codice seguente è necessario per decodificare
			SFP2 . filtro . filtro [ 0 ] = filt [ 0 ];
			SFP2 . filtro . mascherare [ 0 ] = maschera [ 0 ];
			SFP2 . filtro . filtro [ 1 ] = 0 ;
			SFP2 . filtro . Maschera [ 1 ] = 0 ;
			SFP2 . filtro . filtro [ 2 ] = 0 ;
			SFP2 . filtro . maschera [ 2 ] = 0 ;
			memcpy ( SFP2 . filtro . filtrare + 3 , filt + 1 , 16 - 3 );
			memcpy ( SFP2 . filtro . mascherare + 3 , maschera + 1 , 16 - 3 );
			// DeepThought: nei driver delle immagini DGS / CubeStation e Neumo,
			// Dvbapi 1 e 3 sono in qualche modo misto. Nei driver del kernel, il DMX_SET_FILTER
			// Ioctl si aspetta di ricevere una struttura dmx_sct_filter_params (dvbapi 3), ma
			// A causa di un bug relativi insiemi la "maschera positiva" a torto (che dovrebbero essere tutti 0).
			// D'altra parte, l'ioctl DMX_SET_FILTER1 utilizza anche i dmx_sct_filter_params
			// Struttura, che è errata (dovrebbe essere dmxSctFilterParams).
			// L'unico modo per farlo bene è quello di chiamare DMX_SET_FILTER1 con l'argomento
			// Atteso da DMX_SET_FILTER. In caso contrario, il parametro timeout non è passato correttamente.
			ret = dvbapi_ioctl (demux [demux_id]. demux_fd [n]. fd , DMX_SET_FILTER1, e SFP2 );
		}
		altro
		{
			memcpy ( SFP2 . filtro . filtro , filt, 16 );
			memcpy ( SFP2 . filtro . Maschera , la maschera, 16 );
			se (cfg. dvbapi_listenport || cfg. dvbapi_boxtype == BOXTYPE_PC_NODMX)
				ret = dvbapi_net_send (DVBAPI_DMX_SET_FILTER, demux [demux_id]. socket_fd , demux_id, n, ( unsigned  char *) e SFP2 );
			altro
				ret = dvbapi_ioctl (demux [demux_id]. demux_fd [n]. fd , DMX_SET_FILTER, e SFP2 );
		}
		rompere ;

	caso DVBAPI_1:
		ret = demux [demux_id]. demux_fd [n]. fd = dvbapi_open_device ( 0 , demux [demux_id]. demux_index , demux [demux_id]. adapter_index );
		se (ret < 0 ) { ritorno ret; }   // restituisce se dispositivo cant essere aperta!
		struct dmxSctFilterParams SFP1 ;

		memset (& SFP1 , 0 , sizeof ( SFP1 ));

		SFP1 . pid             = pid;
		SFP1 . Timeout         = timeout;
		SFP1 . flags           = DMX_IMMEDIATE_START;
		memcpy ( SFP1 . filtro . filtro , filt, 16 );
		memcpy ( SFP1 . filtro . Maschera , la maschera, 16 );
		ret = dvbapi_ioctl (demux [demux_id]. demux_fd [n]. fd , DMX_SET_FILTER1, e SFP1 );

		rompere ;
# ifdef WITH_STAPI
	caso STAPI:
		ret = stapi_set_filter (. demux_id, pid, filt, maschera, n, demux [demux_id] pmt_file );
		se (ret! = 0 )
			. {Demux [demux_id] demux_fd [n]. fd = ret; }
		altro
			{Ret = - 1 ; } // impostazione filtro errore!
		rompere ;
# endif
# ifdef WITH_COOLAPI
	caso COOLAPI:
		. demux [demux_id] demux_fd [n]. fd = coolapi_open_device (demux [demux_id]. demux_index , demux_id);
		se (demux [demux_id]. demux_fd [n]. fd > 0 )
			{Ret = coolapi_set_filter (. demux [demux_id] demux_fd [n]. fd , n, pid, filt, maschera, tipo); }
		rompere ;
# endif
	predefinito :
		rompere ;
	}
	se (ret = -! 1 )   // filtro scelto di successo
	{
		cs_log_dbg (D_DVBAPI, " Demuxer % d Filtra % d ha iniziato con successo (caid % 04X provid % 06X pid % 04X ) " , demux_id, n + 1 , caid, provid, pid);
		se (tipo == TYPE_EMM && add_to_emm_list) {
			add_emmfilter_to_list (demux_id, filt, caid, provid, pid, n + 1 , true );
		}
	}
	altro
	{
		cs_log ( " ERRORE: Impossibile avviare il filtro demux (api: % d errno = % d  % s ) " , selected_api, ermo, strerror (errno));
	}
	ritorno ret;
}

statica  int32_t  dvbapi_detect_api ( vuoto )
{
# ifdef WITH_COOLAPI
	selected_api = COOLAPI;
	selected_box = 5 ;
	disable_pmt_files = 1 ;
	. cfg dvbapi_listenport = 0 ;
	cs_log ( " Rilevato CoolStream API " );
	ritorno  1 ;
# altro
	se (cfg. dvbapi_boxtype == BOXTYPE_PC_NODMX || cfg. dvbapi_boxtype == BOXTYPE_PC) {
		selected_api = DVBAPI_3;
		selected_box = 1 ;
		se (cfg. dvbapi_listenport )
		{
			cs_log ( " Utilizzo di TCP ascolta presa, API costretto a DVBAPIv3 ( % d ), UserConfig boxtype: % d " ., selected_api, cfg dvbapi_boxtype );
		}
		altro
		{
			cs_log ( " Utilizzo % s ascolto socket API costretto a DVBAPIv3 ( % d ), UserConfig boxtype: % d " ., dispositivi [selected_box] cam_socket_path ., selected_api, cfg dvbapi_boxtype );
		}
		ritorno  1 ;
	}
	altro
	{
		. cfg dvbapi_listenport = 0 ;
	}
	
	int32_t i = 0 , n = 0 , devnum = - 1 , dmx_fd = 0 , ret = 0 , boxnum = sizeof (dispositivi) / sizeof ( struct box_devices);
	char device_path [ 128 ], device_path2 [ 128 ];

	mentre (i <boxnum)
	{
		snprintf (device_path2, sizeof (device_path2), dispositivi [i]. demux_device , 0 );
		snprintf (device_path, sizeof (device_path), dispositivi [i]. percorso , n);
		strncat (device_path, device_path2, sizeof (device_path) - strlen (device_path) - 1 );
		// FIXME: * QUESTO SAMYGO CONTROLLO è testato *
		// FIXME: Rileva samygo, controllando se i percorsi di periferica DVBAPI_3 predefiniti sono prese
		se (i == 1 ) { // Dobbiamo boxnum 1 solo
			struct stat sb;
			se ( stat (device_path, e sb)> 0 && S_ISSOCK (sb. st_mode )) {
				selected_box = 0 ;
				disable_pmt_files = 1 ;
				is_samygo = 1 ;
				devnum = i;
				rompere ;
			}
		}
		se ((dmx_fd = aperto (device_path, O_RDWR | O_NONBLOCK))> 0 )
		{
			devnum = i;
			ret = close (dmx_fd);
			rompere ;
		}
		/ * Provare almeno 8 adattatori * /
		se (( strchr (dispositivi [i]. percorso , ' % ' ) =! NULL ) && (n < 8 )) n ++; altrimenti {n = 0 ; i ++; }
	}

	se (devnum == - 1 ) { ritorno  0 ; }
	selected_box = devnum;
	se (selected_box> - 1 )
		. {Selected_api = dispositivi [selected_box] api ; }
	
	se (ret < 0 ) { cs_log ( " ERRORE: Impossibile chiudere demuxer fd (errno = % d  % s ) " , errno, strerror (errno)); } // il login qui dato che alcuni var necessari non sono Inited prima!
	se (is_samygo) { cs_log ( " SAMYGO rilevato. " ); } // il login qui dato che alcuni var necessari non sono Inited prima!
# ifdef WITH_STAPI
	se (devnum == 4 && stapi_open () == 0 )
	{
		cs_log ( " ERRORE: Stapi: la creazione di Stapi fallito. " );
		ritorno  0 ;
	}
# endif
	se (cfg. dvbapi_boxtype == BOXTYPE_NEUMO)
	{
		selected_api = DVBAPI_3; // DeepThought
	}
	cs_log ( " Rilevato % s Api: % d , UserConfig boxtype: % d " ., device_path, selected_api, cfg dvbapi_boxtype );
# endif
	ritorno  1 ;
}

statica  int32_t  dvbapi_read_device ( int32_t dmx_fd, unsigned  char * buf, int32_t lunghezza)
{
	int32_t len, rc;
	struct pollfd pfd [ 1 ];

	pfd [ 0 ]. fd = dmx_fd;
	pfd [ 0 ]. eventi = (POLLIN | POLLPRI);

	rc = sondaggio (PFD, 1 , 7000 );
	se (rc < 1 )
	{
		cs_log ( " ERRORE: Continuate a leggere % d scaduta (errore = % d  % s ) " , dmx_fd, ermo, strerror (errno));
		ritorno - 1 ;
	}

	len = lettura (dmx_fd, buf, lunghezza);
		
	se (len < 1 )
	{ 
		se (errno == EOVERFLOW)
		{
			cs_log ( " fd % d dati validi presenti dal ricevitore ha riportato un bufferoverflow interna! " , dmx_fd);
			ritorno  0 ;
		}
		altro
		{
			cs_log ( " ERROR: Errore di lettura su fd % d (errno = % d  % s ) " , dmx_fd, ermo, strerror (errno));
		}
	}
	altro { cs_log_dump_dbg (D_TRACE, buf, len, " readed: " ); }
	ritorno len;
}

int32_t  dvbapi_open_device ( int32_t tipo, int32_t num, int32_t adattatore)
{
	int32_t dmx_fd, ret;
	int32_t ca_offset = 0 ;
	char device_path [ 128 ], device_path2 [ 128 ];

	se (cfg. dvbapi_listenport || cfg. dvbapi_boxtype == BOXTYPE_PC_NODMX)
		ritorno DUMMY_FD;
	
	se (tipo == 0 )
	{
		snprintf (device_path2, sizeof (device_path2), i dispositivi [selected_box]. demux_device , num);
		snprintf (device_path, sizeof (device_path), i dispositivi [selected_box]. percorso , adattatore);

		strncat (device_path, device_path2, sizeof (device_path) - strlen (device_path) - 1 );
	}
	altro
	{
		se (cfg. dvbapi_boxtype == BOXTYPE_DUCKBOX || cfg. dvbapi_boxtype == BOXTYPE_DBOX2 || cfg. dvbapi_boxtype == BOXTYPE_UFS910)
			{Ca_offset = 1 ; }

		se (cfg. dvbapi_boxtype == BOXTYPE_QBOXHD)
			{Num = 0 ; }

		se (cfg. dvbapi_boxtype == BOXTYPE_PC)
			{Num = 0 ; }

		snprintf (device_path2, sizeof (device_path2), i dispositivi [selected_box]. ca_device , num + ca_offset);
		snprintf (device_path, sizeof (device_path), i dispositivi [selected_box]. percorso , adattatore);

		strncat (device_path, device_path2, sizeof (device_path) - strlen (device_path) - 1 );
	}

	se (is_samygo) {
		struct sockaddr_un saddr;
		memset (& saddr, 0 , sizeof (saddr));
		saddr. sun_family = AF_UNIX;
		strncpy (. saddr sun_path , device_path, sizeof (saddr. sun_path ) - 1 );
		dmx_fd = Presa (AF_UNIX, SOCK_STREAM, 0 );
		ret = collegare (dmx_fd, ( struct sockaddr *) e saddr, sizeof (saddr));
		se (ret < 0 )
			vicino (dmx_fd);
	} altro {
		dmx_fd = ret = aperto (device_path, O_RDWR | O_NONBLOCK);
	}

	se (ret < 0 )
	{
		cs_log ( " ERRORE: Impossibile aprire il device % s (errno = % d  % s ) " , device_path, ermo, strerror (errno));
		ritorno - 1 ;
	}

	cs_log_dbg (D_DVBAPI, " Aperto dispositivo % s (fd % d ) " , device_path, dmx_fd);

	ritorno dmx_fd;
}

uint16_t  tunemm_caid_map ( uint8_t diretta, uint16_t caid, uint16_t srvid)
{
	int32_t i;
	struct s_client * cl = cur_client ();
	TUNTAB * TTAB = & Cl> TTAB;

	se (! ttab-> ttnum)
		ritorno caid;

	se (diretta)
	{
		per (i = 0 ; i <ttab-> ttnum; i ++)
		{
			se (caid == ttab-> ttdata [i]. bt_caidto
					&& (Srvid == ttab-> ttdata [i]. bt_srvid || ttab-> ttdata [i]. bt_srvid == 0xFFFF ||! ttab-> ttdata [i]. bt_srvid ))
				{ tornare ttab-> ttdata [i]. bt_caidfrom ; }
		}
	}
	altro
	{
		per (i = 0 ; i <ttab-> ttnum; i ++)
		{
			se (caid == ttab-> ttdata [i]. bt_caidfrom
					&& (Srvid == ttab-> ttdata [i]. bt_srvid || ttab-> ttdata [i]. bt_srvid == 0xFFFF ||! ttab-> ttdata [i]. bt_srvid ))
				{ tornare ttab-> ttdata [i]. bt_caidto ; }
		}
	}
	ritorno caid;
}

int32_t  dvbapi_stop_filter ( int32_t demux_index, int32_t tipo)
{
	int32_t g, ret = - 1 ;

	per (g = 0 ; g <MAX_FILTER; g ++) // semplicemente fermarli tutti, noi non vogliamo rischiare di lasciare qualsiasi filtro stantii in esecuzione a causa di abbassamento della maxfilters
	{
		se (demux [demux_index]. demux_fd [g]. tipo == tipo)
		{
			ret = dvbapi_stop_filternum (demux_index, g);
		}
	}
	se (ret == - 1 ) { ritorno  0 ; }   // in caso di errore di ritorno 0
	altro { ritorno  1 ; }
}

int32_t  dvbapi_stop_filternum ( int32_t demux_index, int32_t num)
{
	int32_t retfilter = - 1 , retfd = - 1 ., fd = demux [demux_index] demux_fd [num]. fd ;
	se (fd> 0 )
	{
		cs_log_dbg (D_DVBAPI, " Demuxer % d arresto del filtro % d (fd: % d api: % d , caid: % 04X , provid: % 06X , % s pid: % 04X ) " ,
					  demux_index, num + 1 , fd, selected_api, demux [demux_index]. demux_fd [num]. caid , demux [demux_index]. demux_fd [num]. provid ,
					  (. Demux [demux_index] demux_fd [num]. Tipo == TYPE_ECM? " ecm " : " emm " ), demux [demux_index]. demux_fd [num]. pid );

		interruttore (selected_api)
		{
		caso DVBAPI_3:
			se (cfg. dvbapi_listenport || cfg. dvbapi_boxtype == BOXTYPE_PC_NODMX)
				retfilter = dvbapi_net_send (. DVBAPI_DMX_STOP, demux [demux_index] socket_fd , demux_index, num, NULL );
			altro
				retfilter = dvbapi_ioctl (fd, DMX_STOP, NULL );
			rompere ;

		caso DVBAPI_1:
			retfilter = dvbapi_ioctl (fd, DMX_STOP, NULL );
			rompere ;

# ifdef WITH_STAPI
		caso STAPI:
			retfilter = stapi_remove_filter (. demux_index, num, demux [demux_index] pmt_file );
			se (retfilter! = 1 )    // STAPI restituisce 0 per errore, 1 per tutti ok
			{
				retfilter = - 1 ;
			}
			rompere ;
# endif
# ifdef WITH_COOLAPI
		caso COOLAPI:
			retfilter = coolapi_remove_filter (fd, num);
			retfd = coolapi_close_device (fd);
			rompere ;
# endif
		predefinito :
			rompere ;
		}
		se (retfilter < 0 )
		{
			cs_log ( " ERRORE: Demuxer % d non riusciva a smettere di filtro % d (fd: % d api: % d errno = % d  % s ) " , demux_index, num + 1 , fd, selected_api, ermo, strerror (errno));
		}
# ifndef WITH_COOLAPI // no chiudere fd per coolapi e Stapi, tutti gli altri fanno vicino fd!
		se (! cfg. dvbapi_listenport && cfg. dvbapi_boxtype ! = BOXTYPE_PC_NODMX)
		{
			retfd = close (fd);
			se (errno == 9 ) {retfd = 0 ; }   // nessun errore sul descrittore di file male
			se (selected_api == STAPI) {retfd = 0 ; }   // Stapi chiude la propria fd filtro!
		}
		altro
			retfd = 0 ;
# endif
		se (retfd)
		{
			cs_log ( " ERRORE: Demuxer % d non poteva chiudere fd del filtro % d (fd = % d api: % d errno = % d  % s ) " , demux_index, num + 1 , fd,
				   selected_api, errno, strerror (errno));
		}

		se (. demux [demux_index] demux_fd . [num] tipo == TYPE_ECM)    // filtro ecm fermato: azzerato indice!
		{
			int32_t . idx = demux [demux_index] ECMpids [. demux [demux_index] demux_fd [num]. pidindex ]. Indice ;
			. demux [demux_index] ECMpids [. demux [demux_index] demux_fd [num]. pidindex ]. Indice = 0 ;
			int32_t i;
			per (i = 0 ; i <DEMUX [demux_index]. STREAMpidcount && IDX; i ++) {
				dvbapi_set_pid (demux_index, i, idx - 1 , falso ); // disabilitare tutti streampids per questo indice!
			}
		}

		se (demux [demux_index]. demux_fd [num]. Tipo == TYPE_EMM)    // Se il tipo emm togliere dal emm filterlist
		{
			remove_emmfilter_from_list (. demux_index, demux [demux_index] demux_fd . [num] caid ., demux [demux_index] demux_fd . [num] provid ., demux [demux_index] demux_fd . [num] pid , num + 1 );
		}
		. demux [demux_index] demux_fd [num]. fd = 0 ;
		. demux [demux_index] demux_fd . [num] tipo = 0 ;
	}
	se (retfilter < 0 ) { ritorno retfilter; }   // errore remove filter
	se (retfd < 0 ) { ritorno retfd; }   // errore sul filtro fine fd
	ritornare  1 ; // tutto ok!
}

vuoto  dvbapi_start_filter ( int32_t demux_id, int32_t pidindex, uint16_t pid, uint16_t caid, uint32_t provid, tavolo uchar, maschera uchar, int32_t timeout, int32_t tipo)
{
	uchar filtrare [ 32 ];
	memset (filtro, 0 , 32 );

	filtrare [ 0 ] = tavolo;
	filtrata [ 16 ] = maschera;

	cs_log_dbg (D_DVBAPI, " Demuxer % d tenta di avviare nuovo filtro per caid: % 04X , provid: % 06X , PID: % 04X " , demux_id, caid, provid, pid);
	dvbapi_set_filter (demux_id, selected_api, pid, caid, provid, filtrante, filtro + 16 , timeout, pidindex, tipo, 0 );
}

vuoto  dvbapi_start_emm_filter ( int32_t demux_index)
{
	unsigned  int j;
	se (! demux [demux_index]. EMMpidcount )
		{ ritorno ; }

	// If (demux [demux_index] .emm_filter)
	// Restituisce;


	struct s_csystem_emm_filter * dmx_filter = NULL ;
	unsigned  int filter_count = 0 ;
	uint16_t caid, ncaid;
	uint32_t provid;

	struct s_reader * rdr = NULL ;
	struct s_client * cl = cur_client ();
	se (! cl ||! Cl> aureader_list)
		{ ritorno ; }

	LL_ITER itr = ll_iter_create (Cl> aureader_list);
	mentre ((rdr = ll_iter_next (& ITR)))
	{
		se (RDR-> audisabled ||! RDR-> abilitare || (! is_network_reader (rdr) && RDR-> card_status! = CARD_INSERTED))
			{ continua ; }

		const  struct s_cardsystem * csystem;
		uint16_t c, partita;
		cs_log_dbg (D_DVBAPI, " Demuxer % d lettore corrispondenza % s contro emmpids disponibili -> INIZIO! " , demux_index, RDR-> etichetta);
		per (c = 0 ;. c <demux [demux_index] EMMpidcount ; c ++)
		{
			caid = ncaid = demux [demux_index]. EMMpids [c]. CAID ;
			se (Caid!) continua ;

			se ( chk_is_betatunnel_caid (caid) == 2 )
			{
				ncaid = tunemm_caid_map (. from_to, caid, demux [demux_index] program_number );
			}
			provid = demux [demux_index]. EMMpids [c]. provid ;
			se (Caid == ncaid)
			{
				partita = emm_reader_match (rdr, caid, provid);
			}
			altro
			{
				partita = emm_reader_match (rdr, ncaid, provid);
			}
			se (partita)
			{
				csystem = get_cardsystem_by_caid (caid);
				se (csystem)
				{
					se (Caid! = ncaid)
					{
						csystem = get_cardsystem_by_caid (ncaid);
						se (csystem && csystem-> get_tunemm_filter)
						{
							csystem-> get_tunemm_filter (rdr, e dmx_filter, e filter_count);
							cs_log_dbg (D_DVBAPI, " Demuxer % d filtro impostazione emm per betatunnel: % 04X -> % 04X " , demux_index, ncaid, caid);
						}
						altro
						{
							cs_log_dbg (D_DVBAPI, " Demuxer % d cardsystem per il filtro emm per caid % 04X del lettore % s non trovato " , demux_index, ncaid, RDR-> etichetta);
							continua ;
						}
					}
					altrimenti  se (csystem-> get_emm_filter)
					{
						csystem-> get_emm_filter (rdr, e dmx_filter, e filter_count);
					}
				}
				altro
				{
					cs_log_dbg (D_DVBAPI, " Demuxer % d cardsystem per il filtro emm per caid % 04X del lettore % s non trovato " , demux_index, caid, RDR-> etichetta);
					continua ;
				}

				per (j = 0 ; j <filter_count; j ++)
				{
					se (dmx_filter [j]. enabled == 0 )
					{ continua ; }

					uchar filtrare [ 32 ];
					memset (filtro, 0 , sizeof (filtro));   // più scelte
					uint32_t usefilterbytes = 16 ; // uso di default tutti i filtri
					memcpy (. filtro, dmx_filter [j] filtro , usefilterbytes);
					memcpy (filtro + 16 , dmx_filter [j]. maschera , usefilterbytes);
					int32_t . emmtype = dmx_filter [j] tipo ;

					se (filtro [ 0 ] && ((( 1 << (filtro [ 0 ]% 0x80)) e RDR-> b_nano) &&! (( 1 << (filtro [ 0 ]% 0x80)) e RDR-> s_nano) ))
					{ 
						cs_log_dbg (D_DVBAPI, " Demuxer % d lettore % s emmfilter % d / % d bloccato da UserConfig -> SKIP! " , demux_index, RDR-> etichetta, j + 1 , filter_count);
						continua ;
					}

					se ((RDR-> blockemm & emmtype) &&! ((( 1 << (filtro [ 0 ]% 0x80)) e RDR-> s_nano) || (RDR-> saveemm & emmtype)))
					{ 
						cs_log_dbg (D_DVBAPI, " Demuxer % d lettore % s emmfilter % d / % d bloccato da UserConfig -> SKIP! " , demux_index, RDR-> etichetta, j + 1 , filter_count);
						continua ;
					}
				
					se (demux [demux_index]. EMMpids [c]. Tipo & emmtype)
					{
						cs_log_dbg (D_DVBAPI, " Demuxer % d lettore % s emmfilter % d / % d tipo di corrispondenza -> ATTIVA! " , demux_index, RDR-> etichetta, j + 1 , filter_count);
						check_add_emmpid (demux_index, filtro, c, emmtype);
					}
					altro
					{
						cs_log_dbg (D_DVBAPI, " Demuxer % d lettore % s emmfilter % d / % d tipo non corrispondente -> SKIP! " , demux_index, RDR-> etichetta, j + 1 , filter_count);
					}
				}
					
				// Non dmx_filter utilizzare sotto di questo punto;
				NULLFREE (dmx_filter);
			}
		}
		cs_log_dbg (D_DVBAPI, " Demuxer % d corrispondenza lettore % s contro emmpids disponibili -> FATTO! " , demux_index, RDR-> etichetta);
	}
	se (. demux [demux_index] emm_filter == - 1 ) // innanzitutto eseguire -1
	{
		. demux [demux_index] emm_filter = 0 ;
	}
	cs_log_dbg (D_DVBAPI, " Demuxer % d gestisce % i emm filtri " , demux_index, demux [demux_index]. emm_filter );
}

vuoto  dvbapi_add_ecmpid_int ( int32_t demux_id, uint16_t caid, uint16_t ecmpid, uint32_t provid)
{
	int32_t n, ha aggiunto = 0 ;

	se (== ecmpid 0 ) {
		// DeepThought: collegare a zero pid a pid tunnel
		// (Hack per Nagra / seca tunneling su canaal Vlaanderen Digitaal e tv)
		int i;
		per (i = 0 ; i <demux [demux_id]. ECMpidcount ; i ++)
			se ((demux [demux_id]. ECMpids [i]. PROVID & 0xff) == provid) {
				. ecmpid = demux [demux_id] ECMpids [i]. ECM_PID ;
				se (ecmpid) {
					cs_log ( " Demuxer % d , DT: mappato da 0 a 0x % x \ n " , demux_id, ecmpid);
					rompere ;
				}
			}
	}
	
	se (demux [demux_id]. ECMpidcount > = ECM_PIDS)
		{ ritorno ; }

	int32_t flusso = demux [demux_id]. STREAMpidcount - 1 ;
	per (n = 0 ;. n <demux [demux_id] ECMpidcount ; n ++)
	{
		se (stream> - 1 . && demux [demux_id] ECMpids . [n] CAID . == caid && demux [demux_id] ECMpids . [n] ECM_PID . == ecmpid && demux [demux_id] ECMpids . [n] provid == provid)
		{
			se (! demux [demux_id]. ECMpids [n]. flussi )
			{
				// Abbiamo già ottenuto questo caid / ecmpid come globale, non c'è bisogno di aggiungere il singolo flusso
				cs_log ( " Demuxer % d saltato flusso CAID: % 04X ECM_PID: % 04X PROVID: % 06X (Uguale a ECMPID % d ) " , demux_id, caid, ecmpid, provid, n);
				continua ;
			}
			aggiunto = 1 ;
			. demux [demux_id] ECMpids [n]. torrenti | = ( 1 << stream);
			cs_log ( " Demuxer % d aggiunto al flusso ecmpid % d CAID: % 04X ECM_PID: % 04X PROVID: % 06X " , demux_id, n, caid, ecmpid, provid);
		}
	}

	se (== aggiunto 1 )
		{ ritorno ; }
	per (n = 0 ;. n <demux [demux_id] ECMpidcount ; n ++)   // verificare pid esistente
	{
		se (demux [demux_id]. ECMpids [n]. CAID == caid && demux [demux_id]. ECMpids [n]. ECM_PID == ecmpid && demux [demux_id]. ECMpids [n]. provid == provid)
			{ ritorno ; } // trovati stesso pid -> saltare
	}
	. demux [demux_id] ECMpids [demux [demux_id]. ECMpidcount ]. ECM_PID = ecmpid;
	. demux [demux_id] ECMpids [demux [demux_id]. ECMpidcount ]. CAID = caid;
	demux [demux_id]. ECMpids [demux [demux_id]. ECMpidcount ]. PROVID = provid;
	. demux [demux_id] ECMpids [. demux [demux_id] ECMpidcount .] CHID = 0x10000; // resettare CHID
	. demux [demux_id] ECMpids [demux [demux_id]. ECMpidcount ]. controllato = 0 ;
	//demux[demux_id].ECMpids[demux[demux_id].ECMpidcount].index = 0;
	. demux [demux_id] ECMpids [. demux [demux_id] ECMpidcount .] stato = 0 ;
	. demux [demux_id] ECMpids [. demux [demux_id] ECMpidcount .] cerca = 0xFE;
	. demux [demux_id] ECMpids [. demux [demux_id] ECMpidcount .] ruscelli = 0 ; // azzerare i flussi!
	. demux [demux_id] ECMpids [demux [demux_id]. ECMpidcount ]. irdeto_curindex = 0xFE; // Reset
	. demux [demux_id] ECMpids [demux [demux_id]. ECMpidcount ]. irdeto_curindex = 0xFE; // Reset
	. demux [demux_id] ECMpids [demux [demux_id]. ECMpidcount ]. irdeto_maxindex = 0 ; // resettare
	. demux [demux_id] ECMpids [demux [demux_id]. ECMpidcount ]. irdeto_cycle = 0xFE; // Reset
	. demux [demux_id] ECMpids [. demux [demux_id] ECMpidcount .] table = 0 ;

	se (stream> - 1 )
		. {Demux [demux_id] ECMpids [. demux [demux_id] ECMpidcount .] flussi | = ( 1 << stream); }

	cs_log ( " Demuxer % d ha aggiunto una nuova ecmpid % d CAID: % 04X ECM_PID: % 04X PROVID: % 06X " ., demux_id, demux [demux_id] ECMpidcount , caid, ecmpid, provid);
	se ( caid_is_irdeto . (Caid)) {demux [demux_id] emmstart . tempo = 1 ; }   // marcatore per recuperare emms presto Irdeto ha bisogno di loro!

	. demux [demux_id] ECMpidcount ++;
}

vuoto  dvbapi_add_ecmpid ( int32_t demux_id, uint16_t caid, uint16_t ecmpid, uint32_t provid)
{;
	dvbapi_add_ecmpid_int (demux_id, caid, ecmpid, provid);
	struct s_dvbapi_priority * joinentry;

	per (joinentry = dvbapi_priority;! joinentry = null ; joinentry = joinentry-> successiva)
	{
		se ((joinentry-> digitare! = ' j ' )
				|| (Joinentry-> caid && joinentry-> caid! = Caid)
				|| (Joinentry-> provid && joinentry-> provid! = Provid)
				|| (Joinentry-> ecmpid && joinentry-> ecmpid! = Ecmpid)
				|| (Joinentry-> srvid && joinentry-> srvid! = Demux [demux_id]. program_number ))
			{ continua ; }
		cs_log_dbg (D_DVBAPI, " Join ecmpid % 04X : % 06X : % 04X a 04X% : % 06X : % 04X " ,
					  caid, provid, ecmpid, joinentry-> mapcaid, joinentry-> mapprovid, joinentry-> mapecmpid);
		dvbapi_add_ecmpid_int (demux_id, joinentry-> mapcaid, joinentry-> mapecmpid, joinentry-> mapprovid);
	}
}

vuoto  dvbapi_add_emmpid ( int32_t demux_id, uint16_t caid, uint16_t emmpid, uint32_t provid, uint8_t tipo)
{
	char TypeText [ 40 ];
	cs_strncpy (TypeText, " : " , sizeof (TypeText));

	se (tipo e 0x01) { strcat (TypeText, " UNICA: " ); }
	se (tipo e 0x02) { strcat (TypeText, " CONDIVISA: " ); }
	se (tipo e 0x04) { strcat (TypeText, " GLOBAL: " ); }
	se (tipo e 0xF8) { strcat (TypeText, " UNKNOWN: " ); }

	uint16_t i;
	per (i = 0 ; i <demux [demux_id]. EMMpidcount ; i ++)
	{
		se (demux [demux_id]. EMMpids [i]. PID == emmpid && demux [demux_id]. EMMpids [i]. CAID == caid && demux [demux_id]. EMMpids [i]. provid == provid)
		{
			se (! (demux [demux_id]. EMMpids [i]. tipo e tipo)) {
				. demux [demux_id] EMMpids . [i] tipo | = tipo; // registrare questo tipo emm a questo emmpid
				cs_log_dbg (D_DVBAPI, " In aggiunta a emmpid esistente % d emmtype ulteriore % s " demux [demux_id],. EMMpidcount - 1 , TypeText);
			}
			ritorno ;
		}
	}
	demux [demux_id]. EMMpids [demux [demux_id]. EMMpidcount ]. PID = emmpid;
	demux [demux_id]. EMMpids [demux [demux_id]. EMMpidcount ]. CAID = caid;
	demux [demux_id]. EMMpids [demux [demux_id]. EMMpidcount ]. PROVID = provid;
	demux [demux_id]. EMMpids [demux [demux_id]. EMMpidcount ++]. Tipo = tipo;
	cs_log_dbg (D_DVBAPI, " Aggiunto nuovo emmpid % d CAID: % 04X EMM_PID: % 04X PROVID: % 06X TIPO % s " ., demux [demux_id] EMMpidcount - 1 , caid, emmpid, provid, TypeText);
}

vuoto  dvbapi_parse_cat ( int32_t demux_id, uchar * buf, int32_t len)
{
# ifdef WITH_COOLAPI
	// Autista a volte riporta errore se troppi filtri emm
	// Ma l'aggiunta più filtro ecm non è un problema
	// ... Così ifdef qui invece di limitare MAX_FILTER
	. demux [demux_id] max_emm_filter = 14 ;
# altro
	se (cfg. dvbapi_requestmode == 1 )
	{
		uint16_t ecm_filter_needed = 0 , n;
		per (n = 0 ;. n <demux [demux_id] ECMpidcount ; n ++)
		{
			se (. demux [demux_id] ECMpids . [n] stato > - 1 )
				{Ecm_filter_needed ++; }
		}
		se (maxfilter - ecm_filter_needed <= 0 )
			. {Demux [demux_id] max_emm_filter = 0 ; }
		altro
			{Demux [demux_id]. max_emm_filter = maxfilter - ecm_filter_needed; }
	}
	altro
	{
		. demux [demux_id] max_emm_filter = maxfilter - 1 ;
	}
# endif
	uint16_t i, k;

	cs_log_dump_dbg (D_DVBAPI, buf, len, " gatto: " );

	per (i = 8 ; i <( B2i ( 2 , BUF + 1 ) e 0xFFF) - 1 ; i + = buf [i + 1 ] + 2 )
	{
		se (buf [i] = 0x09!) { continua ; }
		se (demux [demux_id]. EMMpidcount > = ECM_PIDS) { rompere ; }

		uint16_t Caid = B2i ( 2 , buf + i + 2 );
		uint16_t emm_pid = B2i ( 2 , buf + i + 4 ) e 0x1FFF;
		uint32_t emm_provider = 0 ;

		interruttore (Caid >> 8 )
		{
			cassa 0x01:
				dvbapi_add_emmpid (demux_id, caid, emm_pid, 0 , EMM_UNIQUE | EMM_GLOBAL);
				per (k = i + 7 ; k <i + buf [i + 1 ] + 2 ; k + = 4 )
				{
					emm_provider = B2i ( 2 , buf + k + 2 );
					emm_pid = B2i ( 2 , buf + k) & 0xFFF;
					dvbapi_add_emmpid (demux_id, caid, emm_pid, emm_provider, EMM_SHARED);
				}
				rompere ;
			cassa 0x05:
				per (k = i + 6 ; k <i + buf [i + 1 ] + 2 ; k + = buf [k + 1 ] + 2 )
				{
					se (buf [k] == 0x14)
					{
						emm_provider = ( B2i ( 3 , buf + k + 2 ) e 0xFFFFF0); // viaccess correzione ultima cifra è una cura di Non!
						dvbapi_add_emmpid (demux_id, caid, emm_pid, emm_provider, EMM_UNIQUE | EMM_SHARED | EMM_GLOBAL);
					}
				}
				rompere ;
			cassa 0x18:
				se (buf [i + 1 ] == 0x07 || buf [i + 1 ] == 0x0B)
				{
					per (k = i + 7 ; k <i + 7 + buf [i + 6 ]; k + = 2 )
					{
						emm_provider = B2i ( 2 , buf + k);
						dvbapi_add_emmpid (demux_id, caid, emm_pid, emm_provider, EMM_UNIQUE | EMM_SHARED | EMM_GLOBAL);
					}
				}
				altro
				{
					dvbapi_add_emmpid (demux_id, caid, emm_pid, emm_provider, EMM_UNIQUE | EMM_SHARED | EMM_GLOBAL);
				}
				rompere ;
			predefinito :
				dvbapi_add_emmpid (demux_id, caid, emm_pid, 0 , EMM_UNIQUE | EMM_SHARED | EMM_GLOBAL);
				rompere ;
		}
	}
	ritorno ;
}

statica  pthread_mutex_t lockindex;
int32_t  dvbapi_get_descindex ( int32_t demux_index)
{
	int32_t i, j, idx = 1 , fallire = 1 ;
	se (cfg. dvbapi_boxtype == BOXTYPE_NEUMO)
	{
		idx = 0 ;
		sscanf (demux [demux_index]. pmt_file , " pmt % 3d .tmp " , e IDX);
		idx ++; // correzione
		ritorno idx;
	}
	pthread_mutex_lock (& lockindex); // per evitare gara quando i lettori diventano sensibili!
	mentre (non riuscito)
	{
		fail = 0 ;
		per (i = 0 ; i <MAX_DEMUX; i ++)
		{
			per (j = 0 ; j <demux [i]. ECMpidcount ; j ++)
			{
				se (demux [i]. ECMpids [j]. Indice == idx)
				{
					idx ++;
					fail = 1 ;
					rompere ;
				}
			}
		}
	}
	pthread_mutex_unlock (& lockindex); // e rilasciarlo!
	ritorno idx;
}

vuoto  dvbapi_set_pid ( int32_t demux_id, int32_t num, int32_t idx, bool abilitare)
{
	int32_t i, currentfd;
	// If (demux [demux_id] .pidindex == -1) return;

	interruttore (selected_api)
	{
# ifdef WITH_STAPI
	caso STAPI:
		se (! abilitare) idx = - 1 ;
		stapi_set_pid (demux_id, num, idx, demux [demux_id]. STREAMpids [num], demux [demux_id]. pmt_file ); // Utilizzato solo per disabilitare pid !!!
		rompere ;
# endif
# ifdef WITH_COOLAPI
	caso COOLAPI:
		rompere ;
# endif
	predefinito :
		per (i = 0 ; i <MAX_DEMUX; i ++)
		{
			se (demux [demux_id]. ca_mask & ( 1 << i))
			{	
				int8_t action = 0 ;
				se (attiva) {
					action = update_streampid_list (i, demux [demux_id]. STREAMpids [num], idx);
				}
				se (! abilitare) {
					action = remove_streampid_from_list (i, demux [demux_id]. STREAMpids [num], idx);
				}
				se (azione! = NO_STREAMPID_LISTED azione &&! = FOUND_STREAMPID_INDEX)
				{
					ca_pid_t ca_pid2;
					memset (& ca_pid2, 0 , sizeof (ca_pid2));
					ca_pid2. pid = demux [demux_id]. STREAMpids [num];
					se (azione == REMOVED_STREAMPID_LASTINDEX) idx = - 1 ; // rimosso ultimo indice di streampid -> disable pid con -1
					. ca_pid2 index = idx;

					cs_log_dbg (D_DVBAPI, " Demuxer % d  % s flusso % d pid = 0x % 04x index = % d ca % d " , demux_id,
						(? Abilitare " abilitare " : " disattivare " ), num + 1 , ca_pid2. pid , ca_pid2. Indice , i);

					se (cfg. dvbapi_boxtype == BOXTYPE_PC || cfg. dvbapi_boxtype == BOXTYPE_PC_NODMX)
						dvbapi_net_send (. DVBAPI_CA_SET_PID, demux [demux_id] socket_fd , demux_id, - 1  / * inutilizzato * / , ( unsigned  char *) e ca_pid2);
					altro
					{
						currentfd = ca_fd [i];
						se (currentfd <= 0 )
						{
							currentfd = dvbapi_open_device ( 1 , i, demux [demux_id]. adapter_index );
							ca_fd [i] = currentfd; // risparmio fd di questo ca
						}
						se (currentfd> 0 )
						{
							se ( dvbapi_ioctl (currentfd, CA_SET_PID, e ca_pid2) == - 1 )
								cs_log_dbg (D_TRACE | D_DVBAPI, " errore CA_SET_PID ioctl (errno = % d  % s ) " , errno, strerror (errno));
							int8_t risultato = is_ca_used (i);
							se (! consentire && risultato == CA_IS_CLEAR) {
								cs_log_dbg (D_DVBAPI, " Demuxer % d vicino ora CA inutilizzato % d dispositivo " , demux_id, i);
								int32_t ret = close (currentfd);
								se (ret < 0 ) { cs_log ( " ERRORE: Impossibile chiudere demuxer fd (errno = % d  % s ) " , errno, strerror (errno)); }
								currentfd = ca_fd [i] = 0 ;
							}
						}
					}
				}
			}
		}
		rompere ;
	}
	ritorno ;
}

vuoto  dvbapi_stop_descrambling ( int32_t demux_id)
{
	int32_t i;
	se (. demux [demux_id] program_number == 0 ) { ritorno ; }
	char channame [ 32 ];
	i = demux [demux_id]. pidindex ;
	se (i < 0 ) {i = 0 ; }
	int32_t . idx = demux [demux_id] ECMpids . [i] indice ;
	get_servicename (. dvbapi_client, demux [demux_id] program_number , demux [demux_id]. ECMpidcount > 0 demux [demux_id]. ECMpids [i]. CAID : 0 , channame);
	cs_log_dbg (D_DVBAPI, " Demuxer % d numero di programma fermata ricomporre % 04X ( % s ) " , demux_id, demux [demux_id]. program_number , channame);
	dvbapi_stop_filter (demux_id, TYPE_EMM);
	se (demux [demux_id]. ECMpidcount > 0 )
	{
		dvbapi_stop_filter (demux_id, TYPE_ECM);
		demux [demux_id]. pidindex = - 1 ;
		demux [demux_id]. curindex = - 1 ;
		per (i = 0 ; i <demux [demux_id]. STREAMpidcount ; i ++)
		{
			dvbapi_set_pid (demux_id, i, idx - 1 , falso ); // disabilitare tutti streampids per questo indice!
		}
	}

	memset (& demux [demux_id], 0 , sizeof (DEMUXTYPE));
	demux [demux_id]. pidindex = - 1 ;
	demux [demux_id]. curindex = - 1 ;
	se (! cfg. dvbapi_listenport && cfg. dvbapi_boxtype ! = BOXTYPE_PC_NODMX)
		unlink (ECMINFO_FILE);
	ritorno ;
}

int32_t  dvbapi_start_descrambling ( int32_t demux_id, int32_t pid, int8_t selezionata)
{
	int32_t iniziato = 0 ; // in caso ecmfilter iniziato = 1
	int32_t fake_ecm = 0 ;
	ECM_REQUEST * ER;
	struct s_reader * rdr;
	se ((er =! get_ecmtask {())) ritorno iniziato; }
	demux [demux_id]. ECMpids [pid]. controllato = controllato + 1 ; // contrassegnare questo pid come bagaglio!

	struct s_dvbapi_priority p *;
	per (p = dvbapi_priority; p =! NULL ; p = p-> successiva)
	{
		se ((p-> digitare! = ' p ' )
				|| (P-> caid && p-> caid! = Demux [demux_id]. ECMpids [pid]. CAID )
				|| (P-> provid && p-> provid! = Demux [demux_id]. ECMpids [pid]. PROVID )
				|| (P-> ecmpid && p-> ecmpid! = Demux [demux_id]. ECMpids [pid]. ECM_PID )
				|| (P-> srvid && p-> srvid! = Demux [demux_id]. program_number )
				|| (P-> PIDX && p-> pidx- 1 ! = pid))
			{ continua ; }
		// Se chid trovata e prima esecuzione applicare il filtro chid, sul pid forzati sempre applicare!
		se (p-> digitare == ' p ' && p-> chid <0x10000 && (demux [demux_id]. ECMpids [pid]. controllato == 1 || (p && p-> forza)))
		{
			se (demux [demux_id]. ECMpids [pid]. CHID <0x10000)    // channelcache chid consegnato
			{
				ER-> chid = demux [demux_id]. ECMpids [pid]. CHID ;
			}
			altro
			{
				ER-> chid = p-> chid; // no channelcache o no chid in uso, in modo da utilizzare prio chid
				. demux [demux_id] ECMpids . [pid] CHID = p-> chid;
			}
			// Cs_log ("********* CHID% 04X **************" demux [demux_id] .ECMpids [pid] .CHID);
			spezziamo ; // accettiamo soltanto uno!
		}
		altro
		{
			se (demux [demux_id]. ECMpids [pid]. CHID <0x10000)    // channelcache chid consegnato
			{
				ER-> chid = demux [demux_id]. ECMpids [pid]. CHID ;
			}
			altrimenti    // no channelcache o no chid in uso
			{
				ER-> chid = 0 ;
				. demux [demux_id] ECMpids . [pid] CHID = 0x10000;
			}
		}
	}
	ER-> srvid = demux [demux_id]. program_number ;
	. ER-> caid = demux [demux_id] ECMpids [pid]. CAID ;
	. ER-> pid = demux [demux_id] ECMpids [pid]. ECM_PID ;
	ER-> prid = demux [demux_id]. ECMpids [pid]. provid ;
	. ER-> vpid = demux [demux_id] ECMpids [pid]. VPID ;
	ER-> pmtpid = demux [demux_id]. pmtpid ;

	struct voltaB ora;
	cs_ftime (e ora);
	per (rdr = first_active_reader;! rdr = null ; rdr = RDR-> successiva)
	{
		int8_t partita = matching_reader (ehm, rdr); // verificare la corrispondenza lettore
		int64_t andato = comp_timeb (e ora, e RDR-> emm_last);
		se (andata> 3600 * 1000 && RDR-> needsemmfirst && caid_is_irdeto (ER-> Caid))
		{
			cs_log ( " Attenzione lettore % s non ha ricevuto emms per gli ultimi % d secondi -> salta, questo lettore deve emms prima! " , RDR-> etichetta,
				   ( int ) (andata / 1000 ));
			continuare ; // saltare questa carta deve elaborare emms prima prima di poter essere utilizzato per Decodifica
		}
		se (p && p-> forza) {match = 1 ; }   // costretto PID sempre iniziare!

		se (partita!) // se questo lettore non corrisponde, controlla betatunnel per esso
			partita = lb_check_auto_betatunnel (ehm, rdr);

		se (! incontro && chk_is_betatunnel_caid (ER-> Caid))   // questi caids potrebbe essere tunnel invisibile da coetanei
			{Match = 1 ; } // in modo da fare una partita di provarlo!

		se ( config_enabled (CS_CACHEEX) && (! partita && ( cacheex_is_match_alias (dvbapi_client, ehm))))    // verificare se la cache-ex è la corrispondenza
		{
			partita = 1 ; // in modo da fare una partita di provarlo!
		}

		// BISS CAID o FALSO
		// Flusso ecm pid è falso, quindi inviare una richiesta ecm falso
		// Trattamento speciale: se abbiamo chiesto il cw prima senza avviare un filtro la richiesta cw sarà ucciso a causa della mancanza ecmfilter iniziato
		se ( caid_is_fake (demux [demux_id]. ECMpids [pid]. CAID ) || caid_is_biss (demux [demux_id]. ECMpids [pid]. CAID ))
		{
			int32_t j, n;
			ER-> ecmlen = 5 ;
			ER-> ecm [ 0 ] = 0x80; // per passare la cache controllare deve essere 0x80 o ​​0x81
			ER-> ecm [ 1 ] = 0x00;
			ER-> ecm [ 2 ] = 0x02;
			i2b_buf ( 2 , er-> srvid, ehm> ecm + 3 );

			per (j = 0 , n = 5 ;. j <demux [demux_id] STREAMpidcount ; j ++, n + = 2 )
			{
				i2b_buf ( 2 ., demux [demux_id] STREAMpids [j], er-> ECM + n);
				er-> ECM [ 2 ] + = 2 ;
				ER-> ecmlen + = 2 ;
			}

			cs_log ( " Demuxer % d cercando di decodificare PID % d CAID % 04X PROVID % 06X ECMPID % 04X QUALSIASI CHID PMTPID % 04X VPID % 04X " , demux_id, pid,
				   demux [demux_id]. ECMpids [pid]. CAID , demux [demux_id]. ECMpids [pid]. provid , demux [demux_id]. ECMpids [pid]. ECM_PID ,
				   . demux [demux_id] pmtpid ., demux [demux_id] ECMpids [pid]. VPID );

			demux [demux_id]. curindex = pid; // corrente impostata pid al fresco iniziato uno

			dvbapi_start_filter (demux_id, pid, demux [demux_id]. ECMpids [pid]. ECM_PID , demux [demux_id]. ECMpids [pid]. CAID ,
								. demux [demux_id] ECMpids [pid]. PROVID , 0x80, 0xF0, 3000 , TYPE_ECM);
			iniziato = 1 ;

			request_cw (dvbapi_client, ehm, demux_id, 0 ); // non registrare ecm da questa prova!
			fake_ecm = 1 ;
			spezziamo ; // abbiamo iniziato un ecmfilter così smettere di cercare la prossima lettore corrispondenza!
		}
		se (partita)    // se il controllo di lettore trovato per irdeto cas se locale assegno carta irdeto se ha ricevuto emms in ultimi 60 minuti
		{

			se ( caid_is_irdeto (ER-> Caid))    // irdeto cas init irdeto_curindex aspettare per primo indice (00)
			{
				se (demux [demux_id]. ECMpids [pid]. irdeto_curindex == 0xFE) {demux [demux_id]. ECMpids [pid]. irdeto_curindex = 0x00; }
			}

			se (p && p-> chid <0x10000)   // facciamo Prio una certa chid?
			{
				cs_log ( " Demuxer % d cercando di decodificare PID % d CAID % 04X PROVID % 06X ECMPID % 04X CHID % 04X PMTPID % 04X VPID % 04X " , demux_id, pid,
					   demux [demux_id]. ECMpids [pid]. CAID , demux [demux_id]. ECMpids [pid]. provid , demux [demux_id]. ECMpids [pid]. ECM_PID ,
					   . demux [demux_id] ECMpids . [PID] CHID ., demux [demux_id] pmtpid ., demux [demux_id] ECMpids [pid]. VPID );
			}
			altro
			{
				cs_log ( " Demuxer % d cercando di decodificare PID % d CAID % 04X PROVID % 06X ECMPID % 04X QUALSIASI CHID PMTPID % 04X VPID % 04X " , demux_id, pid,
					   demux [demux_id]. ECMpids [pid]. CAID , demux [demux_id]. ECMpids [pid]. provid , demux [demux_id]. ECMpids [pid]. ECM_PID ,
					   . demux [demux_id] pmtpid ., demux [demux_id] ECMpids [pid]. VPID );
			}

			demux [demux_id]. curindex = pid; // corrente impostata pid al fresco iniziato uno

			dvbapi_start_filter (demux_id, pid, demux [demux_id]. ECMpids [pid]. ECM_PID , demux [demux_id]. ECMpids [pid]. CAID ,
								. demux [demux_id] ECMpids [pid]. PROVID , 0x80, 0xF0, 3000 , TYPE_ECM);
			iniziato = 1 ;
			spezziamo ; // abbiamo iniziato un ecmfilter così smettere di cercare la prossima lettore corrispondenza!
		}
	}
	se (demux [demux_id]. curindex ! = pid)
	{
		cs_log ( " Demuxer % d impossibile decodificare PID % d CAID % 04X PROVID % 06X ECMPID % 04X PMTPID % 04X (NO LETTORE MATCHING) " , demux_id, pid,
			   . demux [demux_id] ECMpids . [pid] CAID ., demux [demux_id] ECMpids . [pid] provid ., demux [demux_id] ECMpids [pid]. ECM_PID , demux [demux_id]. pmtpid );
		. demux [demux_id] ECMpids [pid]. controllò = 4 ; // segnalare questo pid come verificato
		. demux [demux_id] ECMpids . [pid] stato = - 1 ; // segnalare questo pid come inutilizzabile
		dvbapi_edit_channel_cache (demux_id, pid, 0 ); // rimuovere questo pid da channelcache
	}
	se (fake_ecm!) { NULLFREE (er); }
	ritorno iniziato;
}

struct s_dvbapi_priority * dvbapi_check_prio_match_emmpid ( int32_t demux_id, uint16_t caid, uint32_t provid, char tipo)
{
	struct s_dvbapi_priority p *;
	int32_t i;

	uint16_t ecm_pid = 0 ;
	per (i = 0 ; i <demux [demux_id]. ECMpidcount ; i ++)
	{
		se ((demux [demux_id]. ECMpids [i]. CAID == caid) && (demux [demux_id]. ECMpids [i]. provid == provid))
		{
			. ecm_pid = demux [demux_id] ECMpids [i]. ECM_PID ;
			rompere ;
		}
	}

	se (ecm_pid!)
		{ ritorno  NULL ; }

	per (p = dvbapi_priority; p =! NULL ; p = p-> successiva)
	{
		se (p-> digitare! = tipo
				|| (P-> caid && p-> caid! = Caid)
				|| (P-> provid && p-> provid! = Provid)
				|| (P-> ecmpid && p-> ecmpid! = Ecm_pid)
				|| (P-> srvid && p-> srvid! = Demux [demux_id]. program_number )
				|| (P-> PIDX && p-> pidx- 1 ! = i)
				|| (P-> tipo == ' i ' && (p-> chid <0x10000)))
			{ continua ; }
		ritorno p;
	}
	ritorno  NULL ;
}

struct s_dvbapi_priority * dvbapi_check_prio_match ( int32_t demux_id, int32_t pidindex, char tipo)
{
	struct s_dvbapi_priority p *;
	struct s_ecmpids * ecmpid = & demux [demux_id]. ECMpids [pidindex];

	per (p = dvbapi_priority; p =! NULL ; p = p-> successiva)
	{
		se (p-> digitare! = tipo
				|| (P-> caid && p-> caid! = Ecmpid-> CAID)
				|| (P-> provid && p-> provid! = Ecmpid-> PROVID)
				|| (P-> ecmpid && p-> ecmpid! = Ecmpid-> ECM_PID)
				|| (P-> srvid && p-> srvid! = Demux [demux_id]. program_number )
				// || (P-> tipo == 'i' && (p-> chid> -1))) /// ????
				|| (P-> PIDX && p-> pidx- 1 ! = pidindex)
				|| (P-> chid <0x10000 && p-> chid! = Ecmpid-> CHID))
			{ continua ; }
		ritorno p;
	}
	ritorno  NULL ;
}

vuoto  dvbapi_process_emm ( int32_t demux_index, int32_t filter_num, unsigned  char * buffer, uint32_t len)
{
	EMM_PACKET EPG;

	cs_log_dbg (D_DVBAPI, " Demuxer % d Filtrare % d dati emm inverosimile " , demux_index, filter_num + 1 ); // EMM mostrato con -d64

	struct s_emm_filter * filter = get_emmfilter_by_filternum (demux_index, filter_num + 1 );

	se (! filtro)
	{
		cs_log_dbg (D_DVBAPI, " Demuxer % d Filter % d ! alcun filtro corrisponde -> SKIP " , demux_index, filter_num + 1 );
		ritorno ;
	}

	uint32_t provider = Filtro-> provid;
	uint16_t caid = Filtro-> caid;

	struct s_dvbapi_priority * mapentry = dvbapi_check_prio_match_emmpid (Filtro-> demux_id, Filtro-> caid, Filtro-> provid, ' m ' );
	se (mapentry)
	{
		cs_log_dbg (D_DVBAPI, " Demuxer % d mappatura EMM da % 04X : % 06X a 04X% : % 06X " , demux_index, caid, fornitore, mapentry-> mapcaid,
					  mapentry-> mapprovid);
		caid = mapentry-> mapcaid;
		provider = mapentry-> mapprovid;
	}

	memset (& EPG, 0 , sizeof (EPG));

	i2b_buf ( 2 , caid, EPG. caid );
	i2b_buf ( 4 , erogatori, EPG. provid );

	EPG. emmlen = len> sizeof (EPG. emm )? sizeof (EPG. emm ): len;
	memcpy (. EPG emm , tampone, EPG. emmlen );

	se ( config_enabled (READER_IRDETO) && chk_is_betatunnel_caid (caid) == 2 )
	{
		uint16_t ncaid = tunemm_caid_map (. from_to, caid, demux [demux_index] program_number );
		se (Caid! = ncaid)
		{
			irdeto_add_emm_header (e EPG);
			i2b_buf ( 2 , ncaid, EPG. caid );
		}
	}

	do_emm (dvbapi_client, e EPG);
}

vuoto  dvbapi_read_priority ( vuoto )
{
	FILE * fp;
	char gettone [ 128 ], str1 [ 128 ];
	char tipo;
	int32_t i, ret, count = 0 ;

	const  char * cs_prio = " oscam.dvbapi " ;

	fp = fopen ( get_config_filename (token, sizeof (token), cs_prio), " r " );

	se (! fp)
	{
		cs_log_dbg (D_DVBAPI, " ERRORE: Impossibile priorità aprire il file % s " , token);
		ritorno ;
	}

	se (dvbapi_priority)
	{
		cs_log_dbg (D_DVBAPI, " rileggere la priorità del file % s " , cs_prio);
		struct s_dvbapi_priority * o, * p;
		per (p = dvbapi_priority;! p = null ; p = o)
		{
			o = p-> next;
			NULLFREE (p);
		}
		dvbapi_priority = NULL ;
	}

	mentre ( fgets (gettone, sizeof (token), fp))
	{
		// Ignorare commenti e le righe vuote
		se (token [ 0 ] == ' # ' || gettone [ 0 ] == ' / ' || gettone [ 0 ] == ' \ n ' || gettone [ 0 ] == ' \ r ' || segno [ 0 ] == ' \ 0 ' )
			{ continua ; }
		se ( strlen (token)> 100 ) { continua ; }

		memset (str1, 0 , 128 );

		per (i = 0 ; i <( int ) strlen (token) && gettone [i] == '  ' ; i ++) {; }
		se (i == ( int ) strlen (token) - 1 )   Struttura // riga vuota o tutti
			{ continua ; }

		per (i = 0 ; i <( int ) strlen (token); i ++)
		{
			se ((token [i] == ' : ' || segno [i] == '  ' ) && gettone [i + 1 ] == ' : ' )   // se "::" o ":"
			{
				memmove (token + i + 2 , segno + i + 1 , strlen (token) - i + 1 ); // inserire la posizione supplementare
				gettone [i + 1 ] = ' 0 ' ; // e riempirlo con NULL
			}
			se (token [i] == ' # ' || gettone [i] == ' / ' )
			{
				gettone [i] = ' \ 0 ' ;
				rompere ;
			}
		}

		type = 0 ;
# ifdef WITH_STAPI
		uint32_t DisableFilter = 0 ;
		ret = sscanf ( assetto (token), " % c : % 63s  % 63s  % d " , e il tipo, str1, str1 + 64 , e DisableFilter);
# altro
		ret = sscanf ( assetto (token), " % c : % 63s  % 63s " , e il tipo, str1, str1 + 64 );
# endif
		type = tolower ((uchar) tipo);

		se (ret < 1 || (tipo! = ' p ' && tipo! = ' i ' && tipo! = ' m ' && tipo! = ' d ' && tipo! = ' s ' && tipo! = ' l '
					   && Tipo! = ' j ' && tipo! = ' un ' && tipo! = ' x ' ))
		{
			// Fprintf (stderr, "Attenzione: riga contenente% s in% s non riconosciuto, ignorando riga \ n", segno, cs_prio);
			// Fprintf avrebbe emesso l'avviso per la linea di comando, che è più coerente con le altre avvertenze di configurazione
			// Ma ci vuole OSCam molto tempo (> 4 secondi) per raggiungere questa parte del programma, in modo che le avvertenze stanno raggiungendo tty piuttosto tardi
			// Che porta a confusione. Quindi invia le avvertenze di file di log invece
			cs_log_dbg (D_DVBAPI, " avvertono: riga contenente % s in % s non riconosciuto, ignorando linea \ n " , segno, cs_prio);
			continua ;
		}

		struct s_dvbapi_priority * ingresso;
		se (! cs_malloc (e la voce, sizeof ( struct s_dvbapi_priority)))
		{
			ret = fclose (fp);
			se (ret < 0 ) { cs_log ( " ERRORE: Impossibile chiudere oscam.dvbapi fd (errno = % d  % s ) " , errno, strerror (errno)); }
			ritorno ;
		}

		entry-> type = tipo;
		entry-> next = NULL ;

		contare ++;

# ifdef WITH_STAPI
		se (tipo == ' s ' )
		{
			strncpy (entry-> devname , str1, 29 );
			strncpy (entry-> pmtfile, str1 + 64 , 29 );

			entry-> DisableFilter = DisableFilter;

			cs_log_dbg (D_DVBAPI, " Stapi PRIO: ret = % d | % c : % s  % s | disabilitare % d " ,
						  ret, tipo, entry-> devname , entry-> pmtfile, DisableFilter);

			se (dvbapi_priority!)
			{
				dvbapi_priority = ingresso;
			}
			altro
			{
				struct s_dvbapi_priority p *;
				per (p = dvbapi_priority; p-> next =! NULL ; p = p-> successiva) {; }
				p-> next = ingresso;
			}
			continua ;
		}
# endif

		char c_srvid [ 34 ];
		c_srvid [ 0 ] = ' \ 0 ' ;
		uint32_t caid = 0 , provid = 0 , srvid = 0 , ecmpid = 0 ;
		uint32_t chid = 0x10000; // chid = 0 è un chid valida
		ret = sscanf (str1, " % 4x : % 6x : % 33 [^:]: % 4x : % 4x " SCNx16, e caid, e provid, c_srvid, e ecmpid, e chid);
		se (ret < 1 )
		{
			cs_log ( " Error in oscam.dvbapi: ret = % d | % c : % 04X  % 06X  % s  % 04X  % 04X " ,
				   ret, tipo, caid, provid, c_srvid, ecmpid, chid);
			continuare ; // saltare questa voce!
		}
		altro
		{
			cs_log_dbg (D_DVBAPI, " regola Parsing: ret = % d | % c : % 04X  % 06X  % s  % 04X  % 04X " ,
						  ret, tipo, caid, provid, c_srvid, ecmpid, chid);
		}

		entry-> caid = caid;
		entry-> provid = provid;
		entry-> ecmpid = ecmpid;
		entry-> chid = chid;

		uint32_t ritardo = 0 , forza = 0 , mapcaid = 0 , mapprovid = 0 , mapecmpid = 0 , PIDX = 0 ;
		Interruttore (tipo)
		{
		caso  ' i ' :
			ret = sscanf (str1 + 64 , " % 1d " , e PIDX);
			entry-> PIDX = PIDX + 1 ;
			se (ret < 1 ) entry-> PIDX = 0 ;
			rompere ;
		caso  ' d ' :
			sscanf (str1 + 64 , " % 4d " , e ritardo);
			entry-> delay = ritardo;
			rompere ;
		caso  ' l ' :
			entry-> ritardo = dyn_word_atob (str1 + 64 );
			se (entry-> ritardo == - 1 ) {entry-> ritardo = 0 ; }
			rompere ;
		caso  ' p ' :
			ret = sscanf (str1 + 64 , " % 1d : % 1d " , e forza, e PIDX);
			entry-> forza = forza;
			entry-> PIDX = PIDX + 1 ;
			se (ret < 2 ) entry-> PIDX = 0 ;
			rompere ;
		caso  ' m ' :
			sscanf (str1 + 64 , " % 4x : % 6x " , e mapcaid, e mapprovid);
			se (mapcaid!) {mapcaid = 0xFFFF; }
			entry-> mapcaid = mapcaid;
			entry-> mapprovid = mapprovid;
			rompere ;
		caso  ' un ' :
		caso  ' j ' :
			sscanf (str1 + 64 , " % 4x : % 6x : % 4x " , e mapcaid, e mapprovid, e mapecmpid);
			se (mapcaid!) {mapcaid = 0xFFFF; }
			entry-> mapcaid = mapcaid;
			entry-> mapprovid = mapprovid;
			entry-> mapecmpid = mapecmpid;
			rompere ;
		}

		se (c_srvid [ 0 ] == ' = ' )
		{
			struct s_srvid * questo;

			per (i = 0 ; i < 16 ; i ++)
				per (questo = cfg. srvid [i];! questo = null ; questo = this-> successiva)
				{
					se ( strcmp (this-> prov, c_srvid + 1 ) == 0 )
					{
						struct s_dvbapi_priority * entry2;
						se (! cs_malloc (& entry2, sizeof ( struct s_dvbapi_priority)))
							{ continua ; }
						memcpy (entry2, ingresso, sizeof ( struct s_dvbapi_priority));

						entry2-> srvid = this-> srvid;

						cs_log_dbg (D_DVBAPI, " PRIO srvid: ret = % d | % c : % 04X  % 06X  % 04X  % 04X  % 04X -> mappa % 04X  % 06X  % 04X | prio % d | ritardo % d " ,
									  ret, entry2-> tipo, entry2-> caid, entry2-> provid, entry2-> srvid, entry2-> ecmpid, entry2-> chid,
									  entry2-> mapcaid, entry2-> mapprovid, entry2-> mapecmpid, entry2-> forza, entry2-> ritardo);

						se (dvbapi_priority!)
						{
							dvbapi_priority = entry2;
						}
						altro
						{
							struct s_dvbapi_priority p *;
							per (p = dvbapi_priority; p-> next =! NULL ; p = p-> successiva) {; }
							p-> next = entry2;
						}
					}
				}
			NULLFREE (voce);
			continua ;
		}
		altro
		{
			sscanf (c_srvid, " % 4x " , e srvid);
			entry-> srvid = srvid;
		}

		cs_log_dbg (D_DVBAPI, " PRIO: ret = % d | % c : % 04X  % 06X  % 04X  % 04X  % 04X -> mappa % 04X  % 06X  % 04X | prio % d | ritardo % d " ,
					  ret, entry-> tipo, entry-> caid, entry-> provid, entry-> srvid, entry-> ecmpid, entry-> chid, entry-> mapcaid,
					  entry-> mapprovid, entry-> mapecmpid, entry-> forza, entry-> ritardo);

		se (dvbapi_priority!)
		{
			dvbapi_priority = ingresso;
		}
		altro
		{
			struct s_dvbapi_priority p *;
			per (p = dvbapi_priority; p-> next =! NULL ; p = p-> successiva) {; }
			p-> next = ingresso;
		}
	}

	cs_log_dbg (D_DVBAPI, " Leggi % d voci % s " , contare, cs_prio);

	ret = fclose (fp);
	se (ret < 0 ) { cs_log ( " ERRORE: Impossibile chiudere oscam.dvbapi fd (errno = % d  % s ) " , errno, strerror (errno)); }
	ritorno ;
}

vuoto  dvbapi_resort_ecmpids ( int32_t demux_index)
{
	int32_t n, cache = 0 , PRIO = 1 , highest_prio = 0 , matching_done = 0 , trovato = - 1 ;
	uint16_t btun_caid = 0 ;
	struct inizio voltaB, fine;
	cs_ftime (& start);
	per (n = 0 ;. n <demux [demux_index] ECMpidcount ; n ++)
	{
		. demux [demux_index] ECMpids . [n] stato = 0 ;
		. demux [demux_index] ECMpids [n]. controllato = 0 ;
	}

	. demux [demux_index] max_status = 0 ;
	demux [demux_index]. curindex = - 1 ;
	demux [demux_index]. pidindex = - 1 ;

	struct s_channel_cache * c = NULL ;

	per (n = 0 ;. n <demux [demux_index] ECMpidcount ; n ++)
	{
		c = dvbapi_find_channel_cache (demux_index, n, 0 ); // trova corrispondenza esatta canale
		se (c! = NULL )
		{
			trovato = n;
			cache = 2 ; // cache trovata con una priorità più alta
			. demux [demux_index] ECMpids . [n] stato = prio * 2 ; // priorità caids che già decodificato stesso caid: provid: srvid
			se (c-> chid <0x10000) {demux [demux_index]. ECMpids [n]. CHID = c-> chid; } // se chid registrata nella cache -> usarlo!
			cs_log_dbg (D_DVBAPI, " Demuxer % d prio ecmpid % d  % 04X : % 06X : % 04X (che si trova caid / provid / srvid nella cache - peso: % d ) " , demux_index, n,
				. demux [demux_index] ECMpids . [n] CAID ., demux [demux_index] ECMpids . [n] provid ., demux [demux_index] ECMpids [n]. ECM_PID , demux [demux_index]. ECMpids [n]. di stato );
			rompere ;
		}
	}
	
	se (== trovati - 1 )
	{
		// Priorità caids che già decodificati stesso caid: provid
		per (n = 0 ;. n <demux [demux_index] ECMpidcount ; n ++)
		{
			c = dvbapi_find_channel_cache (demux_index, n, 1 );
			se (c! = NULL )
			{
				cache = 1 ; // voce della cache trovato
				. demux [demux_index] ECMpids . [n] stato = prio;
				cs_log_dbg (D_DVBAPI, " Demuxer % d prio ecmpid % d  % 04X : % 06X : % 04X (che si trova caid / provid nella cache - peso: % d ) " , demux_index, n,
					. demux [demux_index] ECMpids . [n] CAID ., demux [demux_index] ECMpids . [n] provid ., demux [demux_index] ECMpids [n]. ECM_PID , demux [demux_index]. ECMpids [n]. di stato );
			}
		}
	}

	// Priorità e ignorare secondo oscam.dvbapi e cfg.preferlocalcards
	se (! dvbapi_priority) { cs_log_dbg (D_DVBAPI, " Demuxer % d non oscam.dvbapi trovato o regole vigenti vengono analizzati! " , demux_index); }

	se (dvbapi_priority)
	{
		struct s_reader * rdr;
		ECM_REQUEST * ER;
		se (! cs_malloc (& er, sizeof (ECM_REQUEST)))
			{ ritorno ; }

		int32_t add_prio = 0 ; // assicurarsi che p: valori annullano la cache
		se (cache == 1 )
			{Add_prio = prio; }
		altrimenti  se (cache == 2 )
			{Add_prio = prio * 2 ; }

		// Ordine inverso! fa in modo che definito dall'utente p: i valori sono nel giusto ordine
		int32_t p_order = demux [demux_index]. ECMpidcount ;

		highest_prio = (prio * demux [demux_index]. ECMpidcount ) + p_order;

		struct s_dvbapi_priority p *;
		per (p = dvbapi_priority; p =! NULL ; p = p-> successiva)
		{
			se (p-> digitare! = ' p ' && p-> digitare! = ' i ' )
				{ continua ; }
			per (n = 0 ;. n <demux [demux_index] ECMpidcount ; n ++)
			{
				se (! nascondiglio && demux [demux_index]. ECMpids [n]. stato ! = 0 )
					{ continua ; }
				altrimenti  se (cache == 1 && (demux [demux_index]. ECMpids [n]. stato < 0 || demux [demux_index]. ECMpids [n]. stato > prio))
					{ continua ; }
				altrimenti  se (cache == 2 && (demux [demux_index]. ECMpids [n]. stato < 0 || demux [demux_index]. ECMpids [n]. stato > prio * 2 ))
					{ continua ; }

				. ER-> caid = er-> ocaid = demux [demux_index] ECMpids [n]. CAID ;
				. ER-> prid = demux [demux_index] ECMpids [n]. provid ;
				. ER-> pid = demux [demux_index] ECMpids [n]. ECM_PID ;
				ER-> srvid = demux [demux_index]. program_number ;
				ER-> client = cur_client ();

				btun_caid = chk_on_btun (SRVID_MASK, er-> cliente, ehm);
				se (p-> digitare == ' p ' && btun_caid)
					{Er-> caid = btun_caid; }

				se (p-> caid && p-> caid! = er-> caid)
					{ continua ; }
				se (p-> provid && p-> provid! = er-> prid)
					{ continua ; }
				se (p-> ecmpid && p-> ecmpid! = er-> pid)
					{ continua ; }
				se (p-> srvid && p-> srvid! = er-> srvid)
					{ continua ; }
				se (p-> PIDX && p-> pidx- 1 ! = n)
					{ continua ; }

				se (p-> tipo == ' i ' )     // controllare se ignorata dai dvbapi
				{
					se (p-> chid == 0x10000)    // ignorare tutto? disabilitare pid
					{
						. demux [demux_index] ECMpids . [n] stato = - 1 ;
					}
					cs_log_dbg (D_DVBAPI, " Demuxer % d ignorare ecmpid % d  % 04X : % 06X : % 04X : % 04X (file) " ., demux_index, n, demux [demux_index] ECMpids . [n] CAID ,
								  . demux [demux_index] ECMpids [n]. PROVID , demux [demux_index]. ECMpids [n]. ECM_PID , ( uint16_t ) p-> chid);
					continua ;
				}

				se (p-> digitare == ' p ' )
				{
					se (demux [demux_index]. ECMpids [n]. stato == - 1 )   // saltare ignora
						{ continua ; }

					matching_done = 1 ;
					per (rdr = first_active_reader; rdr; rdr = RDR-> successiva)
					{
						se (CFG. preferlocalcards &&! is_network_reader (rdr)
								&& RDR-> card_status == CARD_INSERTED)    // cfg.preferlocalcards = 1 lettore locale
						{

							se ( matching_reader (ehm, rdr))
							{
								se (cache == 2 && demux [demux_index]. ECMpids [n]. stato == 1 )
									. {Demux [demux_index] ECMpids [n]. stato ++; }
								altrimenti  se (cache &&! demux [demux_index]. ECMpids [n]. stato )
									. {Demux [demux_index] ECMpids . [n] stato + = add_prio; }
								// Priorità * ECMpidcount dovrebbe prevalere lettore di rete
								. demux [demux_index] ECMpids . [n] stato + = (prio * demux [demux_index]. ECMpidcount ) + (p_order--);
								cs_log_dbg (D_DVBAPI, " Demuxer % d prio ecmpid % d  % 04X : % 06X : % 04X : % 04X (localrdr: % s peso: % d ) " , demux_index,
											  n, demux [demux_index]. ECMpids [n]. CAID , demux [demux_index]. ECMpids [n]. PROVID ,
											  demux [demux_index]. ECMpids [n]. ECM_PID , ( uint16_t ) p-> chid, RDR-> etichetta,
											  . demux [demux_index] ECMpids [n]. stato );
								rompere ;
							}
						}
						altro         // cfg.preferlocalcards = 0 o cfg.preferlocalcards = 1 e nessun lettore locali
						{
							se ( matching_reader (ehm, rdr))
							{
								se (cache == 2 && demux [demux_index]. ECMpids [n]. stato == 1 )
									. {Demux [demux_index] ECMpids [n]. stato ++; }
								altrimenti  se (cache &&! demux [demux_index]. ECMpids [n]. stato )
									. {Demux [demux_index] ECMpids . [n] stato + = add_prio; }
								. demux [demux_index] ECMpids . [n] stato + = prio + (p_order--);
								cs_log_dbg (D_DVBAPI, " Demuxer % d prio ecmpid % d  % 04X : % 06X : % 04X : % 04X (rdr: % s peso: % d ) " , demux_index,
											  n, demux [demux_index]. ECMpids [n]. CAID , demux [demux_index]. ECMpids [n]. PROVID ,
											  demux [demux_index]. ECMpids [n]. ECM_PID , ( uint16_t ) p-> chid, RDR-> etichetta,
											  . demux [demux_index] ECMpids [n]. stato );
								rompere ;
							}
						}
					}
				}
			}
		}
		NULLFREE (er);
	}

	se (! matching_done)       // funziona se non c'è oscam.dvbapi o se c'è regole oscam.dvbapi eccetto p in esso
	{
		se (dvbapi_priority &&! matching_done)
			{ cs_log_dbg (D_DVBAPI, " Demuxer % d regole prio in partite oscam.dvbapi! " , demux_index); }

		struct s_reader * rdr;
		ECM_REQUEST * ER;
		se (! cs_malloc (& er, sizeof (ECM_REQUEST)))
			{ ritorno ; }

		highest_prio = prio * 2 ;

		per (n = 0 ;. n <demux [demux_index] ECMpidcount ; n ++)
		{
			se (demux [demux_index]. ECMpids [n]. stato == - 1 )   // saltare ignora
				{ continua ; }

			. ER-> caid = er-> ocaid = demux [demux_index] ECMpids [n]. CAID ;
			. ER-> prid = demux [demux_index] ECMpids [n]. provid ;
			. ER-> pid = demux [demux_index] ECMpids [n]. ECM_PID ;
			ER-> srvid = demux [demux_index]. program_number ;
			ER-> client = cur_client ();

			btun_caid = chk_on_btun (SRVID_MASK, er-> cliente, ehm);
			se (btun_caid)
				{Er-> caid = btun_caid; }

			per (rdr = first_active_reader; rdr; rdr = RDR-> successiva)
			{
				se (CFG. preferlocalcards
						&&! is_network_reader (rdr)
						&& RDR-> card_status == CARD_INSERTED)    // cfg.preferlocalcards = 1 lettore locale
				{
					se ( matching_reader (ehm, rdr))
					{
						. demux [demux_index] ECMpids . [n] stato + = prio * 2 ;
						cs_log_dbg (D_DVBAPI, " Demuxer % d prio ecmpid % d  % 04X : % 06X : % 04X (localrdr: % s peso: % d ) " , demux_index,
									  n, demux [demux_index]. ECMpids [n]. CAID , demux [demux_index]. ECMpids [n]. PROVID ,
									  demux [demux_index]. ECMpids [n]. ECM_PID , RDR-> etichetta,
									  . demux [demux_index] ECMpids [n]. stato );
						rompere ;
					}
				}
				altro         // cfg.preferlocalcards = 0 o cfg.preferlocalcards = 1 e nessun lettore locali
				{
					se ( matching_reader (ehm, rdr))
					{
						. demux [demux_index] ECMpids . [n] stato + = prio;
						cs_log_dbg (D_DVBAPI, " Demuxer % d prio ecmpid % d  % 04X : % 06X : % 04X (rdr: % s peso: % d ) " , demux_index,
									  n, demux [demux_index]. ECMpids [n]. CAID , demux [demux_index]. ECMpids [n]. PROVID ,
									  . demux [demux_index] ECMpids . [n] ECM_PID , RDR-> etichetta, demux [demux_index]. ECMpids [n]. stato );
						rompere ;
					}
				}
			}
		}
		NULLFREE (er);
	}

	se (cache == 1 )
		{Highest_prio + = prio; }
	altrimenti  se (cache == 2 )
		{Highest_prio + = prio * 2 ; };

	highest_prio ++;

	per (n = 0 ;. n <demux [demux_index] ECMpidcount ; n ++)
	{
		int32_t nr;
		SIDTAB * sidtab;
		ECM_REQUEST er;
		er. caid   = demux [demux_index]. ECMpids [n]. CAID ;
		er. prid   = demux [demux_index]. ECMpids [n]. provid ;
		er. srvid = demux [demux_index]. program_number ;

		per (nr = 0 , sidtab = cfg. sidtab ; sidtab; sidtab = sidtab-> prossimo, nr ++)
		{
			se (sidtab-> num_caid | sidtab-> num_provid | sidtab-> num_srvid)
			{
				se ((CFG. dvbapi_sidtabs . no & ((SIDTABBITS) 1 << nr)) && ( chk_srvid_match (& er, sidtab)))
				{
					. demux [demux_index] ECMpids . [n] stato = - 1 ; // ignorare
					cs_log_dbg (D_DVBAPI, " Demuxer % d ignorare ecmpid % d  % 04X : % 06X : % 04X (servizio % s ) pos % d " , demux_index,
								  n, demux [demux_index]. ECMpids [n]. CAID , demux [demux_index]. ECMpids [n]. PROVID ,
								  . demux [demux_index] ECMpids . [n] ECM_PID , sidtab-> etichetta, nr);
				}
				se ((CFG. dvbapi_sidtabs . ok & ((SIDTABBITS) 1 << nr)) && ( chk_srvid_match (& er, sidtab)))
				{
					. demux [demux_index] ECMpids . [n] stato = highest_prio ++; // priorità
					cs_log_dbg (D_DVBAPI, " Demuxer % d prio ecmpid % d  % 04X : % 06X : % 04X (servizio: % s posizione: % d ) " , demux_index,
								  n, demux [demux_index]. ECMpids [n]. CAID , demux [demux_index]. ECMpids [n]. PROVID ,
								  demux [demux_index]. ECMpids [n]. ECM_PID , sidtab-> etichetta,
								  . demux [demux_index] ECMpids [n]. stato );
				}
			}
		}
	}

	struct s_reader * rdr;
	ECM_REQUEST * ER;
	se (! cs_malloc (& er, sizeof (ECM_REQUEST)))
		{ ritorno ; }
	
	per (n = 0 ;. n <demux [demux_index] ECMpidcount ; n ++)
	{
		. ER-> caid = er-> ocaid = demux [demux_index] ECMpids [n]. CAID ;
		. ER-> prid = demux [demux_index] ECMpids [n]. provid ;
		. ER-> pid = demux [demux_index] ECMpids [n]. ECM_PID ;
		ER-> srvid = demux [demux_index]. program_number ;
		ER-> client = cur_client ();
		btun_caid = chk_on_btun (SRVID_MASK, er-> cliente, ehm);
		se (btun_caid)
		{ 
			ER-> caid = btun_caid;
		}
		
		int32_t match = 0 ;
		per (rdr = first_active_reader; rdr; rdr = RDR-> successiva)
		{
			se ( matching_reader (ehm, rdr))
			{
				partita ++;
			}
		}
		se (partita == 0 )
		{
			cs_log_dbg (D_DVBAPI, " Demuxer % d ignorare ecmpid % d  % 04X : % 06X : % 04X : % 04X (nessun lettore matching) " ., demux_index, n, demux [demux_index] ECMpids . [n] CAID ,
				. demux [demux_index] ECMpids . [n] provid ., demux [demux_index] ECMpids [n]. ECM_PID , demux [demux_index]. ECMpids [n]. chid );
			. demux [demux_index] ECMpids . [n] stato = - 1 ;
		}
	}
	NULLFREE (er);
	
	highest_prio = 0 ;
	int32_t highest_priopid = - 1 ;
	per (n = 0 ;. n <demux [demux_index] ECMpidcount ; n ++)
	{
		se (demux [demux_index]. ECMpids [n]. stato > highest_prio) // trovano più alto pid prio
		{ 
			. highest_prio = demux [demux_index] ECMpids [n]. stato ;
			highest_priopid = n;
		}  
		se (. demux [demux_index] ECMpids [n]. stato == 0 ) {demux [demux_index]. ECMpids [n]. controllato = 2 ; }   // imposta pid senza status di no run prio
	}

	struct s_dvbapi_priority * partita;
	per (incontro = dvbapi_priority; abbinare =! NULL ; abbinare = Match-> successiva)
	{
		se (Match-> digitare! = ' p ' )
			{ continua ; }
		se (! || partita! Match-> forza)   // Prio solo valuta forzati di
			{ continua ; }
		per (n = 0 ;. n <demux [demux_index] ECMpidcount ; n ++)
		{
			se (Match-> caid && Match-> caid = demux [demux_index]!. ECMpids [n]. CAID ) { continua ; }
			se (Match-> provid && Match-> provid = demux [demux_index]!. ECMpids [n]. provid ) { continua ; }
			se (Match-> srvid && Match-> srvid = demux [demux_index]!. program_number ) { continua ; }
			se (Match-> ecmpid && Match-> ecmpid = demux [demux_index]!. ECMpids [n]. ECM_PID ) { continua ; }
			se (Match-> PIDX && Match-> pidx- 1 = n!) { continua ; }
			se (Match-> chid <0x10000) {demux [demux_index]. ECMpids [n]. CHID = Match-> chid; }
			. demux [demux_index] ECMpids [n]. stato = ++ highest_prio;
			cs_log_dbg (D_DVBAPI, " Demuxer % d costretto ecmpid % d  % 04X : % 06X : % 04X : % 04X " ., demux_index, n, demux [demux_index] ECMpids . [n] CAID ,
						  . demux [demux_index] ECMpids [n]. PROVID , demux [demux_index]. ECMpids [n]. ECM_PID , ( uint16_t ) Match-> chid);
			demux [demux_index]. max_status = highest_prio; // register maxstatus
			. demux [demux_index] ECMpids [n]. controllato = 0 ; // set costretto pid a correre prio
			Ritorniamo , // accettiamo soltanto uno pid forzata!
		}
	}
	demux [demux_index]. max_status = highest_prio; // register maxstatus
	se (highest_priopid = - 1 && trovato == highest_priopid)      // Trovato nella cache
	{
		per (n = 0 ;. n <demux [demux_index] ECMpidcount ; n ++)
		{
			se (n! = trovati)
			{
				// Disabilitare pid non corrispondenti
				. demux [demux_index] ECMpids . [n] stato = - 1 ;
			}
			altro
			{
				. demux [demux_index] ECMpids . [n] stato = 1 ;
			}
		}
		. demux [demux_index] max_emm_filter = maxfilter - 1 ;
		. demux [demux_index] max_status = 1 ;
		cs_log ( " Demuxer % d trovato canale nella cache e prio corrispondenza -> inizio ricomporre ecmpid % d  " , demux_index, trovato);
	}
	cs_ftime (e fine);
	int64_t andato = comp_timeb (e fine, e iniziare a);
	cs_log_dbg (D_DVBAPI, " Demuxer % d ordinamento dei ecmpids preso % " PRId64 " ms " , demux_index, andato);
	ritorno ;
}

vuoto  dvbapi_parse_descriptor ( int32_t demux_id, uint32_t info_length, unsigned  char * buffer)
{
	// Int32_t ca_pmt_cmd_id = tampone [i + 5];
	uint32_t descriptor_length = 0 ;
	uint32_t j, u;

	se (info_length < 1 )
		{ ritorno ; }

	se (tampone [ 0 ] == 0x01)
	{
		tampone tampone + = 1 ;
		info_length--;
	}

	per (j = 0 ; j <info_length; j + = descriptor_length + 2 )
	{
		descriptor_length = tampone [j + 1 ];

		se (tampone [j] == 0x81 && descriptor_length == 8 )     // descrittore privata di lunghezza 8, assumere enigma / TVH
		{
			. demux [demux_id] enigma_namespace = B2i ( 4 , buffer di + j + 2 );
			. demux [demux_id] tsid = B2i ( 2 , buffer di + j + 6 );
			. demux [demux_id] ONID = B2i ( 2 , buffer di + j + 8 );
			cs_log_dbg (D_DVBAPI, " Trovato tipo PMT: % 02x Lunghezza: % d (supponendo enigma privato descrittore: namespace % 04x tsid % 02x ONID % 02x ) " ,
						  . Buffer [j], descriptor_length, demux [demux_id] enigma_namespace ., demux [demux_id] tsid , demux [demux_id]. ONID );
		}
		altro
		{
			cs_log_dbg (D_DVBAPI, " Trovato tipo PMT: % 02x Lunghezza: % d " , tampone [j], descriptor_length);
		}

		se (tampone [j] = 0x09!) { continua ; }
		se (demux [demux_id]. ECMpidcount > = ECM_PIDS) { rompere ; }

		int32_t descriptor_ca_system_id = B2i ( 2 , buffer di + j + 2 );
		int32_t descriptor_ca_pid = B2i ( 2 , buffer di + j + 4 ) e 0x1FFF;
		int32_t descriptor_ca_provider = 0 ;

		se (descriptor_ca_system_id >> 8 == 0x01)
		{
			per (u = 2 ; u <descriptor_length; u + = 15 )
			{
				descriptor_ca_pid = B2i ( 2 , buffer di + j + u + 2 ) e 0x1FFF;
				descriptor_ca_provider = B2i ( 2 , buffer di + j + u + 4 );
				dvbapi_add_ecmpid (demux_id, descriptor_ca_system_id, descriptor_ca_pid, descriptor_ca_provider);
			}
		}
		altro
		{
			se ( caid_is_viaccess (descriptor_ca_system_id) && descriptor_length == 0x0F && tampone [j + 12 ] == 0x14)
				{Descriptor_ca_provider = B2i ( 3 , tampone + j + 14 ) e 0xFFFFF0; }

			se ( caid_is_nagra (descriptor_ca_system_id) && descriptor_length == 0x07)
				{Descriptor_ca_provider = B2i ( 2 , buffer di + j + 7 ); }
			
			se (descriptor_ca_system_id >> 8 == 0x4A && descriptor_length == 0x05)
				{Descriptor_ca_provider = tampone [j + 6 ]; }

			dvbapi_add_ecmpid (demux_id, descriptor_ca_system_id, descriptor_ca_pid, descriptor_ca_provider);
			
		}
	}

	// Applica mappatura:
	se (dvbapi_priority)
	{
		struct s_dvbapi_priority * mapentry;
		per (j = 0 ; ( int32_t ) j <demux [demux_id]. ECMpidcount ; j ++)
		{
			mapentry = dvbapi_check_prio_match (demux_id, j, ' m ' );
			se (mapentry)
			{
				cs_log_dbg (D_DVBAPI, " Demuxer % d mappatura ecmpid % d da % 04X : % 06X a 04X% : % 06X " , demux_id, j,
							  demux [demux_id]. ECMpids [j]. CAID , demux [demux_id]. ECMpids [j]. PROVID ,
							  mapentry-> mapcaid, mapentry-> mapprovid);
				demux [demux_id]. ECMpids [j]. CAID = mapentry-> mapcaid;
				demux [demux_id]. ECMpids [j]. provid = mapentry-> mapprovid;
			}
		}
	}
}

vuoto  request_cw ( struct s_client * cliente, ECM_REQUEST * ehm, int32_t demux_id, uint8_t delayed_ecm_check)
{
	int32_t filternum = dvbapi_set_section_filter (demux_id, er); // set di filtri ecm per strano -> VisaVersa anche e

	se (! USE_OPENXCAS && filternum < 0 )
	{
		cs_log_dbg (D_DVBAPI, " Demuxer % d non richiede cw -> Filtro ecm stato ucciso! " , demux_id);
		ritorno ;
	}

	cs_log_dbg (D_DVBAPI, " Demuxer % d ottenere controlword! " , demux_id);
	get_cw (client, er);

	se (! USE_OPENXCAS) {
		se (delayed_ecm_check) { memcpy (. demux [demux_id] demux_fd [filternum]. ecmd5 , er-> ecmd5, CS_ECMSTORESIZE); }   // registrare questo ecm come ultima richiesta di questo filtro
		altro { memset (. demux [demux_id] demux_fd [filternum]. ecmd5 , 0 , CS_ECMSTORESIZE); } // azzerare ecmcheck!
	}

# ifdef WITH_DEBUG
	char buf [ECM_FMT_LEN];
	format_ecm (ehm, buf, ECM_FMT_LEN);
	cs_log_dbg (D_DVBAPI, " Demuxer % d richiesta controlword per ecm % s " , demux_id, buf);
# endif
}

vuoto  dvbapi_try_next_caid ( int32_t demux_id, int8_t selezionata)
{

	int32_t n, j, trovati = - 1 , iniziata = 0 ;

	int32_t status = demux [demux_id]. max_status ;

	per (j = Stato; j> = 0 ; j--)     // stato grande prima!
	{

		per (n = 0 ;. n <demux [demux_id] ECMpidcount ; n ++)
		{
			// Cs_log_dbg (D_DVBAPI "Demuxer% d PID% d controllato =% d status =% d (ricerca di pid con stato =% d)", demux_id, n,
			// demux [demux_id] .ECMpids [n] .checked, demux [demux_id] .ECMpids [n] .STATUS, j);
			se (demux [demux_id]. ECMpids [n]. controllati == controllato && demux [demux_id]. ECMpids [n]. stato == j)
			{
				trovato = n;

				openxcas_set_provid (demux. [demux_id] ECMpids [trovato]. provid );
				openxcas_set_caid (demux. [demux_id] ECMpids [trovato]. CAID );
				openxcas_set_ecm_pid (. demux [demux_id] ECMpids [trovate]. ECM_PID );

				// Fixup per cas che necessitano emm prima!
				se ( caid_is_irdeto (demux [demux_id]. ECMpids [trovato]. CAID )) {demux [demux_id]. emmstart . tempo = 0 ; }
				iniziato = dvbapi_start_descrambling (demux_id, trovato, controllato);
				se (. CFG dvbapi_requestmode == 0 && iniziato == 1 ) { ritorno ; }   // in requestmode 0 abbiamo solo cominciamo 1 richiesta ecm al momento
			}
		}
	}

	se (trovati == - 1 && demux [demux_id]. pidindex == - 1 )
	{
		cs_log ( " demuxer % d non lettori idonei scoperto che può essere utilizzato per la decodifica! " , demux_id);
		ritorno ;
	}
}

static  void  getDemuxOptions ( int32_t demux_id, unsigned  char *buffer, uint16_t *ca_mask, uint16_t *demux_index, uint16_t *adapter_index, uint16_t *pmtpid)
{
	* Ca_mask = 0x01, * demux_index = 0x00, * adapter_index = 0x00, * pmtpid = 0x00;

	se (tampone [ 17 ] == 0x82 && cuscinetto [ 18 ] == 0x02)
	{
		// Enigma2
		* Ca_mask = tampone [ 19 ];
		uint32_t demuxid = tampone [ 20 ];
		se (demuxid == 0xFF) demuxid = 0 ; (! 0xFF -> "demux-1" = errore) // tryfix prismcube
		* Demux_index = demuxid;
		se (tampone [ 21 ] == 0x84 && tampone [ 22 ] == 0x02) * pmtpid = B2i ( 2 , tampone + 23 );
		se (tampone [ 25 ] == 0x83 && tampone [ 26 ] == 0x01) * adapter_index = tampone [ 27 ]; // dall'indice codice cahandler.cpp 0x83 dell'adattatore
	}

	se (cfg. dvbapi_boxtype == BOXTYPE_IPBOX_PMT)
	{
		* Ca_mask = demux_id + 1 ;
		* Demux_index = demux_id;
	}

	se (cfg. dvbapi_boxtype == BOXTYPE_QBOXHD && tampone [ 17 ] == 0x82 && Buffer [ 18 ] == 0x03)
	{
		// Ca_mask = tampone [19]; // Con STONE 1.0.4 sempre 0x01
		* Demux_index = tampone [ 20 ]; // con STONE 1.0.4 sempre 0x00
		* Adapter_index = tampone [ 21 ]; // con STONE 1.0.4 indice adattatore può essere 0,1,2
		* Ca_mask = ( 1 << * adapter_index); // utilizzare adapter_index come ca_mask (usato come indice per ca_fd [] array)
	}

	se ((CFG. dvbapi_boxtype == BOXTYPE_PC || cfg. dvbapi_boxtype == BOXTYPE_PC_NODMX) && tampone [ 7 ] == 0x82 && tampone [ 8 ] == 0x02)
	{
		* Demux_index = tampone [ 9 ]; // è sempre 0, ma non si sa mai
		* Adapter_index = tampone [ 10 ]; // indice adattatore può essere 0,1,2
		* Ca_mask = ( 1 << * adapter_index); // utilizzare adapter_index come ca_mask (usato come indice per ca_fd [] array)
	}
}

static  vuoto  dvbapi_capmt_notify ( struct demux_s * DMX)
{
	struct s_client * cl;
	per (cl = first_client-> prossimo, cl; cl = Cl> successiva)
	{
		se ((Cl> tip == ' p ' || Cl> tip == ' r ' ) && Cl> lettore && Cl> reader-> ph. c_capmt )
		{
			struct demux_s * curdemux;
			se ( cs_malloc (& curdemux, sizeof ( struct demux_s)))
			{
				memcpy (curdemux, DMX, sizeof ( struct demux_s));
				add_job (cl, ACTION_READER_CAPMT_NOTIFY, curdemux, sizeof ( struct demux_s));
			}
		}
	}
}

int32_t  dvbapi_parse_capmt ( unsigned  char * buffer, uint32_t lunghezza, int32_t connfd, char pmtfile *)
{
	uint32_t i = 0 , esegue = 0 ;
	int32_t j = 0 ;
	int32_t demux_id = - 1 ;
	uint16_t ca_mask, demux_index, adapter_index, pmtpid;

# definire  LIST_MORE 0x00     applicazione // * CA dovrebbe aggiungere un 'PIU' CAPMT opporsi alla lista e iniziare a ricevere l'oggetto successivo
# definire  LIST_FIRST 0x01    applicazione // * CA dovrebbe cancellare la lista quando si riceve un oggetto 'PRIMA' CAPMT, e iniziare a ricevere l'oggetto successivo
# definire  LIST_LAST 0x02    applicazione // * CA dovrebbe aggiungere un oggetto CAPMT 'ULTIMO' alla lista e iniziare a lavorare con la lista
# definiscono  LIST_ONLY 0x03    // * CA applicazione dovrebbe cancellare la lista quando si riceve una 'SOLO' oggetto CAPMT, e iniziare a lavorare con l'oggetto
# definire  list_add 0x04     applicazione // * CA dovrebbe aggiungere un 'ADD' CAPMT opporsi alla lista attuale e iniziare a lavorare con l'elenco aggiornato
# definire  LIST_UPDATE 0x05 applicazione // * CA deve sostituire una voce della lista con un 'UPDATE' oggetto CAPMT, e iniziare a lavorare con l'elenco aggiornato

# ifdef WITH_COOLAPI
	int32_t ca_pmt_list_management = LIST_ONLY;
# altro
	int32_t ca_pmt_list_management = tampone [ 0 ];
# endif
	uint32_t program_number = B2i ( 2 , del buffer + 1 );
	uint32_t program_info_length = B2i ( 2 , tampone + 4 ) e 0xFFF;
	
	cs_log_dump_dbg (D_DVBAPI, tampone, la lunghezza, " capmt: " );
	cs_log_dbg (D_DVBAPI, " ricevitore manda il comando PMT % d per il canale % 04X " , ca_pmt_list_management, program_number);
	se ((ca_pmt_list_management == LIST_FIRST || ca_pmt_list_management == LIST_ONLY) && pmt_stopmarking == 0 )
	{
		per (i = 0 ; i <MAX_DEMUX; i ++)
		{
			se (demux [i]. program_number == 0 ) { continua ; }   // salta demuxer vuoti
			se (. demux [i] socket_fd ! = connfd) { continua ; }   // passare demuxer appartenenti ad altra connessione ca pmt
			demux [i]. stopdescramble = 1 ; // Selez se non utilizzato di nuovo seguendo oggetti PMT.
			cs_log_dbg (D_DVBAPI, " Marked demuxer % d / % d per fermare decodifica " , i, MAX_DEMUX);
			pmt_stopmarking = 1 ;
		}
	}
	getDemuxOptions (i, tampone, e ca_mask, e demux_index, e adapter_index, e pmtpid);
	cs_log_dbg (D_DVBAPI, " Ricevitore vuole demux srvid % 04X su adattatore % 04X camask % 04X indice % 04X pmtpid % 04X " ,
		program_number, adapter_index, ca_mask, demux_index, pmtpid);
	
	per (i = 0 ; i <MAX_DEMUX; i ++)     // cercare demuxer attuali per l'esecuzione lo stesso programma come quello che abbiamo ricevuto in questo oggetto PMT
	{
		se (demux [i]. program_number == 0 ) { continua ; }
		se (. CFG dvbapi_boxtype == BOXTYPE_IPBOX_PMT) demux_index = i; // correzione per ipbox

		bool full_check = 1 , abbinato = 0 ;
		se ( config_enabled (WITH_COOLAPI) || is_samygo)
			full_check = 0 ;

		se (full_check)
			abbinato = (connfd> 0 && demux [i]. socket_fd == connfd) && demux [i]. program_number == program_number;
		altro
			abbinato = connfd> 0 && demux [i]. program_number == program_number;

		se (in abbinamento)
		{
			se (full_check) {
				se (demux [i]. adapter_index = adapter_index!) continuare ; // forse prossime partite demuxer?
				se (demux [i]. ca_mask = ca_mask!) continuare ; // forse prossime partite demuxer?
				se (demux [i]. demux_index = demux_index!) continuare ; // forse prossime partite demuxer?
			}
			se (ca_pmt_list_management == LIST_UPDATE) {
				cs_log ( " Demuxer % d aggiornamento PMT per la decodifica di srvid % 04X ! " , i, program_number);
			}

			demux_id = i;

# se definito WITH_STAPI || definito WITH_COOLAPI || definito WITH_MCA || definito WITH_AZBOX
			dvbapi_stop_descrambling (i); // fermata ricomporre per tutte le finestre tranne scatole dvbapi base
# altro
			cs_log ( " Demuxer % d continuare decodifica srvid % 04X " , i, demux [i]. program_number );
# endif

			openxcas_set_sid (program_number);

			demux [i]. stopdescramble = 0 ; // Dont Stop demuxer corrente!
			se (demux [demux_id]. ECMpidcount =! 0 ) {corsa = 1 ; }   // fissare per canale cambia da fta a strapazzate		
			rompere ; // senza bisogno di esplorare altre demuxer perché abbiamo una trovata!
		}
	}

	// Interrompere decodificare vecchi demuxer da questa connessione PMT ca che arent più utilizzato
	se ((ca_pmt_list_management == LIST_LAST) || (ca_pmt_list_management == LIST_ONLY))
	{
		per (j = 0 ; j <MAX_DEMUX; j ++)
		{
			se (. demux [j] program_number == 0 ) { continua ; }
			se (demux [j]. stopdescramble == 1 ) { dvbapi_stop_descrambling (j); }   // Interrompere descrambling e rimuovere tutte le voci demuxer non nuova PMT.
		}
	}

	se (== demux_id - 1 )
	{
		per (demux_id = 0 ; demux_id <MAX_DEMUX && demux [demux_id]. program_number > 0 ; demux_id ++) {; }
	}

	se (demux_id> = MAX_DEMUX)
	{
		cs_log ( " ERRORE: nessun id gratuito (MAX_DEMUX) " );
		ritorno - 1 ;
	}
	
	demux [demux_id]. program_number = program_number; // farlo presto perché alcuni elementi prio li usano!

	. demux [demux_id] enigma_namespace = 0 ;
	demux [demux_id]. tsid = 0 ;
	demux [demux_id]. ONID = 0 ;
	demux [demux_id]. pmtpid = pmtpid;

	se (pmtfile)
	{
		cs_strncpy (. demux [demux_id] pmt_file , pmtfile, sizeof (demux [demux_id]. pmt_file ));
	}

	se (program_info_length> 1 && program_info_length <lunghezza)
	{
		dvbapi_parse_descriptor (demux_id, program_info_length - 1 , tampone + 7 );
	}

	uint32_t es_info_length = 0 , vpid = 0 ;
	struct s_dvbapi_priority * addentry;

	per (j = 0 ; j <demux [demux_id]. ECMpidcount ; j ++) {   // controllare pid esistente
		. demux [demux_id] ECMpids . [j] flussi = 0 ; // azzerare i flussi!
	}
	demux [demux_id]. STREAMpidcount = 0 ; // azzerare Posti flussi

	per (i = program_info_length + 6 ; i <lunghezza; i + = es_info_length + 5 )
	{
		int32_t stream_type = tampone [i];
		uint16_t elementary_pid = B2i ( 2 , tampone + i + 1 ) e 0x1FFF;
		es_info_length = B2i ( 2 , tampone + i + 3 ) e 0x0fff;
		cs_log_dbg (D_DVBAPI, " Trovato tipo di flusso: % 02x pid: % 04x Lunghezza: % d " , stream_type, elementary_pid, es_info_length);

		se (demux [demux_id]. STREAMpidcount > = ECM_PIDS)
		{
			rompere ;
		}

		. demux [demux_id] STREAMpids [demux [demux_id]. STREAMpidcount ++] = elementary_pid;
		// Trova e registrare videopid
		se (! vpid && (stream_type == 01 || stream_type == 02 || stream_type == 0x10 || stream_type == 0x1B)) {vpid = elementary_pid; }

		se (es_info_length! = 0 && es_info_length <lunghezza)
		{
			dvbapi_parse_descriptor (demux_id, es_info_length, tampone + i + 5 );
		}
		altro
		{
			per (addentry = dvbapi_priority;! addentry = null ; addentry = addentry-> successiva)
			{
				se (addentry-> digitare! = ' un '
						|| (Addentry-> ecmpid && pmtpid && addentry-> ecmpid = pmtpid!) // ecmpid è abusato per tenere pmtpid in caso di A: regola
						|| (Addentry-> ecmpid &&! Pmtpid && addentry-> ecmpid! = Vpid) // alcuni ricevitori dont avanti pmtpid, utilizzare invece vpid
						|| (Addentry-> srvid! = Demux [demux_id]. program_number ))
					{ continua ; }
				cs_log_dbg (D_DVBAPI, " Aggiunto falso ecmpid % 04X : % 06x : % 04x per il flusso in chiaro su srvid % 04X " , addentry-> mapcaid, addentry-> mapprovid,
					addentry-> mapecmpid, demux [demux_id]. program_number );
				dvbapi_add_ecmpid (demux_id, addentry-> mapcaid, addentry-> mapecmpid, addentry-> mapprovid);
				rompere ;
			}
		}
	}
	per (j = 0 ; j <demux [demux_id]. ECMpidcount ; j ++)
	{
		. demux [demux_id] ECMpids [j]. VPID = vpid; // Registra trovato vpid su tutti ecmpids di questo demuxer
	}
	cs_log ( " Demuxer % d trovato % d ECMpids e % d STREAMpids in PMT " , demux_id, demux [demux_id]. ECMpidcount , demux [demux_id]. STREAMpidcount - 1 );

	getDemuxOptions (demux_id, tampone, e ca_mask, e demux_index, e adapter_index, e pmtpid);
	cs_log ( " Demuxer % d ricevitore vuole demux srvid % 04X su adattatore % 04X camask % 04X indice % 04X pmtpid % 04X " , demux_id,
		   . demux [demux_id] program_number , adapter_index, ca_mask, demux_index, pmtpid);
	demux [demux_id]. adapter_index = adapter_index;
	demux [demux_id]. ca_mask = ca_mask;
	. demux [demux_id] rdr = NULL ;
	demux [demux_id]. demux_index = demux_index;
	demux [demux_id]. socket_fd = connfd;
	. demux [demux_id] stopdescramble = 0 ; // rimuovere contrassegno eliminazione!

	// Togliere dal unassoc_fd quando necessario
	per (j = 0 ; j <MAX_DEMUX; j ++)
			se (unassoc_fd [j] == connfd)
					unassoc_fd [j] = 0 ;

	char channame [ 32 ];
	get_servicename (. dvbapi_client, demux [demux_id] program_number , demux [demux_id]. ECMpidcount > 0 demux [demux_id]. ECMpids [ 0 ]. CAID : NO_CAID_VALUE, channame);
	cs_log ( " Demuxer % d nuovo numero di programma: % 04X ( % s ) [pmt_list_management % d ] " , demux_id, program_number, channame, ca_pmt_list_management);

	dvbapi_capmt_notify (e demux [demux_id]);

	cs_log_dbg (D_DVBAPI, " Demuxer %d demux_index: %2d ca_mask: %02x program_info_length: %3d ca_pmt_list_management %02x " ,
				  . demux_id, demux [demux_id] demux_index ., demux [demux_id] ca_mask , program_info_length, ca_pmt_list_management);

	struct s_dvbapi_priority * xtraentry;
	int32_t k, l, m, xtra_demux_id;

	per (xtraentry = dvbapi_priority;! xtraentry = null ; xtraentry = xtraentry-> successiva)
	{
		se (! xtraentry-> digitare = ' x ' ) { continua ; }

		per (j = 0 ; j <= demux [demux_id]. ECMpidcount ; ++ j)
		{
			se ((xtraentry-> caid && xtraentry-> caid! = demux [demux_id]. ECMpids [j]. CAID )
					|| (Xtraentry-> provid && xtraentry-> provid! = Demux [demux_id]. ECMpids [j]. PROVID )
					|| (Xtraentry-> ecmpid && xtraentry-> ecmpid! = Demux [demux_id]. ECMpids [j]. ECM_PID )
					|| (Xtraentry-> srvid && xtraentry-> srvid! = Demux [demux_id]. program_number ))
				{ continua ; }

			cs_log ( " Mapping ecmpid % 04X : % 06X : % 04X : % 04X per xtra demuxer / ca-devices " , xtraentry-> caid, xtraentry-> provid, xtraentry-> ecmpid, xtraentry-> srvid);

			per (xtra_demux_id = 0 ; xtra_demux_id <MAX_DEMUX && demux [xtra_demux_id]. program_number > 0 ; xtra_demux_id ++)
				{; }

			se (xtra_demux_id> = MAX_DEMUX)
			{
				cs_log ( " Trovato nessun dispositivo demux gratuito per i flussi xtra. " );
				continua ;
			}
			// Copiare nuova demuxer
			getDemuxOptions (demux_id, tampone, e ca_mask, e demux_index, e adapter_index, e pmtpid);
			. demux [xtra_demux_id] ECMpids [ 0 ] = demux [demux_id]. ECMpids [j];
			demux [xtra_demux_id]. ECMpidcount = 1 ;
			demux [xtra_demux_id]. STREAMpidcount = 0 ;
			demux [xtra_demux_id]. program_number = demux [demux_id]. program_number ;
			demux [xtra_demux_id]. pmtpid = demux [demux_id]. pmtpid ;
			demux [xtra_demux_id]. demux_index = demux_index;
			demux [xtra_demux_id]. adapter_index = adapter_index;
			demux [xtra_demux_id]. ca_mask = ca_mask;
			demux [xtra_demux_id]. socket_fd = connfd;
			. demux [xtra_demux_id] stopdescramble = 0 ; // rimuovere contrassegno eliminazione!
			. demux [xtra_demux_id] rdr = NULL ;
			demux [xtra_demux_id]. curindex = - 1 ;

			// Aggiungere i flussi di xtra demux
			per (k = 0 ; k <demux [demux_id]. STREAMpidcount ; ++ k)
			{
				se (! demux [demux_id]. ECMpids [j]. torrenti || demux [demux_id]. ECMpids [j]. torrenti & ( 1 << k))
				{
					. demux [xtra_demux_id] ECMpids [ 0 .] flussi | = ( 1 << demux [xtra_demux_id]. STREAMpidcount );
					. demux [xtra_demux_id] STREAMpids [demux [xtra_demux_id]. STREAMpidcount ] = demux [demux_id]. STREAMpids [k];
					++ Demux [xtra_demux_id]. STREAMpidcount ;

					// Associazioni flusso spostamento in demux normale perché abbiamo rimuoverà il torrente tutto
					per (l = 0 ; l <demux [demux_id]. ECMpidcount ; ++ l)
					{
						per (m = k; m <demux [demux_id]. STREAMpidcount - 1 ; ++ m)
						{
							se (demux [demux_id]. ECMpids [l]. torrenti & ( 1 << (m + 1 )))
							{
								. demux [demux_id] ECMpids . [l] flussi | = ( 1 << m);
							}
							altro
							{
								. demux [demux_id] ECMpids [l]. torrenti & = ~ ( 1 << m);
							}
						}
					}

					// Rimuove associazione flusso dal normale dispositivo demux
					per (l = k; l <demux [demux_id]. STREAMpidcount - 1 ; ++ l)
					{
						demux [demux_id]. STREAMpids [l] = demux [demux_id]. STREAMpids [l + 1 ];
					}
					--demux [demux_id]. STREAMpidcount ;
					--k;
				}
			}

			// Rimuove ecmpid da demuxer normale
			per (k = j; k ​​<demux [demux_id]. ECMpidcount ; ++ k)
			{
				demux [demux_id]. ECMpids [k] = demux [demux_id]. ECMpids [k + 1 ];
			}
			--demux [demux_id]. ECMpidcount ;
			--j;

			se (demux [xtra_demux_id]. STREAMpidcount > 0 )
			{
				dvbapi_resort_ecmpids (xtra_demux_id);
				dvbapi_try_next_caid (xtra_demux_id, 0 );
			}
			altro
			{
				cs_log ( " non ha trovato flussi per xtra demuxer Non iniziare la decodifica aggiuntivo su di esso.. » );
			}

			se (demux [demux_id]. STREAMpidcount < 1 )
			{
				cs_log ( " non ha trovato flussi per demuxer normale Non iniziare la decodifica aggiuntivo su di esso.. » );
				openxcas_set_sid (program_number);
				ritorno xtra_demux_id;
			}
		}
	}

	se (in esecuzione == 0 ) // solo fare l'installazione emm su canali non correre!
	{
		demux [demux_id]. emm_filter = - 1 ; // registrarti inizio corsa emmfilter
		se (cfg. dvbapi_au > 0 && demux [demux_id]. emmstart . tempo == 1 )    // irdeto fetch emm gatto diretta!
		{
			cs_ftime (e demux [demux_id]. emmstart ); // trucco per lasciare inizio emm recupero dopo 30 secondi per accelerare zapping
			dvbapi_start_filter (. demux_id, demux [demux_id] pidindex , 0x001, 0x001, 0x01, 0x01, 0xFF, 0 , TYPE_EMM); // CAT
		}
		altro { cs_ftime (e demux [demux_id]. emmstart ); } // per tutti gli altri caids partenza ritardata!
	}
	
	openxcas_set_sid (program_number);
	
	se (demux [demux_id]. ECMpidcount == 0 ) { // per FTA finisce qui, ma fare la registrazione e parte di ecmhandler poiché non ci saranno ECM ha chiesto!
		se (cfg. usrfileflag ) { cs_statistics (dvbapi_client);} // aggiungi utente log precedente canale + tempo sul canale
		dvbapi_client-> last_srvid = demux [demux_id]. program_number ; // imposta nuovo srvid canale
		dvbapi_client-> last_caid = NO_CAID_VALUE; // canali FTA hanno caid!
		dvbapi_client-> lastswitch = dvbapi_client-> = ultima volta (( time_t *) 0 ); // azzerare idle-Time & ultimo interruttore
		ritorno demux_id;
	}

# se ! WITH_STAPI definito &&! WITH_COOLAPI definito &&! WITH_MCA definito &&! WITH_AZBOX definito
	se (in esecuzione) disable_unused_streampids (demux_id); // disattivare tutti streampids non più in uso
# endif
	se (in esecuzione == 0 )    // solo iniziare demuxer se era in esecuzione
	{
		demux [demux_id]. decodingtries = - 1 ;
		dvbapi_resort_ecmpids (demux_id);
		dvbapi_try_next_caid (demux_id, 0 );
	}
	ritorno demux_id;
}


vuoto  dvbapi_handlesockmsg ( unsigned  char * buffer, uint32_t len, int32_t connfd)
{
	uint32_t val = 0 , size = 0 , I, K;

	per (k = 0 ; k <len; k + = 3 + size + val)
	{
		se (tampone [ 0 + k]! = 0x9F || Buffer [ 1 + k]! = 0x80)
		{
			cs_log_dbg (D_DVBAPI, " sconosciuto comando PMT ricevute: % 02x " , tampone [ 0 + k]);
			rompere ;
		}

		se (k> 0 )
			cs_log_dump_dbg (D_DVBAPI, tampone + k, len - k, " Analisi oggetto successivo PMT (s): " );

		se (tampone [ 3 + k] e 0x80)
		{
			val = 0 ;
			size = del buffer [ 3 + k] & 0x7F;
			per (i = 0 ; i <dimensioni; i ++)
				{Val = (val << 8 ) | tampone [i + 1 + 3 + k]; }
			formato ++;
		}
		altro
		{
			val = tampone [ 3 + k] & 0x7F;
			size = 1 ;
		}
		interruttore (buffer [ 2 + k])
		{
		cassa 0x32:
			dvbapi_parse_capmt (buffer + size + 3 + k, val, connfd, NULL );
			rompere ;
		caso 0x3f:
			// 9F 80 3f 04 83 02 00 <demux index>
			cs_log_dump_dbg (D_DVBAPI, tampone, len, " capmt 3f: " );
			// Ipbox fix
			se (cfg. dvbapi_boxtype == BOXTYPE_IPBOX || cfg. dvbapi_listenport )
			{
				int32_t demux_index = tampone [ 7 + k];
				per (i = 0 ; i <MAX_DEMUX; i ++)
				{
					// 0xFF demux_index è un jolly => demuxer vicino tutti i relativi
					se (demux_index == 0xFF)
					{
						se (demux [i]. socket_fd == connfd)
							dvbapi_stop_descrambling (i);
					}
					altrimenti  se (demux [i]. demux_index == demux_index)
					{
						dvbapi_stop_descrambling (i);
						rompere ;
					}
				}
				se (cfg. dvbapi_boxtype == BOXTYPE_IPBOX)
				{
					// Controllo abbiamo alcuna demux in esecuzione su questo fd
					int16_t execlose = 1 ;
					per (i = 0 ; i <MAX_DEMUX; i ++)
					{
						se (demux [i]. socket_fd == connfd)
						{
							execlose = 0 ;
							rompere ;
						}
					}
					se (execlose)
					{
						int32_t ret = close (connfd);
						se (ret < 0 ) { cs_log ( " ERRORE: Impossibile chiudere PMT fd (errno = % d  % s ) " , errno, strerror (errno)); }
					}
				}
			}
			altro
			{
				se (cfg. dvbapi_pmtmode ! = 6 )
				{
					int32_t ret = close (connfd);
					se (ret < 0 ) { cs_log ( " ERRORE: Impossibile chiudere PMT fd (errno = % d  % s ) " , errno, strerror (errno)); }
				}
			}
			rompere ;
		predefinito :
			cs_log_dbg (D_DVBAPI, " handlesockmsg () comando sconosciuto " );
			cs_log_dump (buffer, len, « comando sconosciuto: " );
			rompere ;
		}
	}
}

int32_t  dvbapi_init_listenfd ( vuoto )
{
	int32_t clilen, listenfd;
	struct sockaddr_un servaddr;

	memset (& servaddr, 0 , sizeof ( struct sockaddr_un));
	servaddr. sun_family = AF_UNIX;
	cs_strncpy (. servaddr sun_path , dispositivi [selected_box]. cam_socket_path , sizeof (servaddr. sun_path ));
	clilen = sizeof (servaddr. sun_family ) + strlen (servaddr. sun_path );

	se (( unlink (dispositivi [selected_box]. cam_socket_path ) < 0 ) && (errno! = ENOENT))
		{ ritorno  0 ; }
	se ((listenfd = presa (AF_UNIX, SOCK_STREAM, 0 )) < 0 )
		{ ritorno  0 ; }
	se ( bind (listenfd, ( struct sockaddr *) e servaddr, clilen) < 0 )
		{ ritorno  0 ; }
	se ( ascoltare (listenfd, 5 ) < 0 )
		{ ritorno  0 ; }

	// Cambia il diritto di accesso sulla camd.socket
	// Questo permetterà di oscam eseguito come root, se necessario
	// E consentono ancora cliente non root per collegare alla presa
	chmod (dispositivi [selected_box]. cam_socket_path , S_IRWXU | S_IRWXG | S_IRWXO);

	ritorno listenfd;
}

int32_t  dvbapi_net_init_listenfd ( vuoto )
{
	int32_t listenfd;
	struct SOCKADDR servaddr;

	memset (& servaddr, 0 , sizeof (servaddr));
	SIN_GET_FAMILY (servaddr) = DEFAULT_AF;
	SIN_GET_ADDR (servaddr) = ADDR_ANY;
	SIN_GET_PORT (servaddr) = htons (( uint16_t .) CFG dvbapi_listenport );

	se ((listenfd = presa (DEFAULT_AF, SOCK_STREAM, 0 )) < 0 )
		{ ritorno  0 ; }

	int32_t opt = 0 ;
# ifdef IPV6SUPPORT
	// Imposta l'opzione socket server per l'ascolto su IPv4 e IPv6 contemporaneamente
	setsockopt (listenfd, IPPROTO_IPV6, IPV6_V6ONLY, ( vuoto *) e optare, sizeof (opzionale));
# endif

	opt = 1 ;
	setsockopt (listenfd, SOL_SOCKET, SO_REUSEADDR, ( vuoto *) e optare, sizeof (opzionale));
	set_so_reuseport (listenfd);

	se ( bind (listenfd, ( struct sockaddr *) e servaddr, sizeof (servaddr)) < 0 )
		{ ritorno  0 ; }
	se ( ascoltare (listenfd, 5 ) < 0 )
		{ ritorno  0 ; }

	ritorno listenfd;
}

statica  pthread_mutex_t event_handler_lock;

vuoto  event_handler ( int32_t  UNUSED (segnale))
{
	struct stat pmt_info;
	char dest [ 1024 ];
	DIR * DIRP;
	struct ingresso dirent, * dp = NULL ;
	int32_t i, pmt_fd;
	uchar mbuf [ 2048 ]; // fix sporco: più grande del buffer necessaria per la modalità di PMT CA 6 con molti canali paralleli per decodificare
	se (! dvbapi_client = cur_client ()) { ritorno ; }

	pthread_mutex_lock (& event_handler_lock);

	se (cfg. dvbapi_boxtype == BOXTYPE_PC || cfg. dvbapi_boxtype == BOXTYPE_PC_NODMX)
		{Pausecam = 0 ; }
	altro
	{
		int32_t standby_fd = aperto (STANDBY_FILE, O_RDONLY);
		pausecam = (standby_fd> 0 )? 1 : 0 ;
		se (standby_fd> 0 )
		{
			int32_t ret = close (standby_fd);
			se (ret < 0 ) { cs_log ( " ERRORE: Impossibile chiudere fd standby (errno = % d  % s ) " , errno, strerror (errno)); }
		}
	}

	se (cfg. dvbapi_boxtype == BOXTYPE_IPBOX || cfg. dvbapi_pmtmode == 1 )
	{
		pthread_mutex_unlock (& event_handler_lock);
		ritorno ;
	}

	per (i = 0 ; i <MAX_DEMUX; i ++)
	{
		se (demux [i]. pmt_file [ 0 ]! = 0 )
		{
			snprintf (dest, sizeof (dest), " % s% s " , TMPDIR, demux [i]. pmt_file );
			pmt_fd = aperto (dest, O_RDONLY);
			se (pmt_fd> 0 )
			{
				se ( fstat (pmt_fd, e pmt_info)! = 0 )
				{
					int32_t ret = close (pmt_fd);
					se (ret < 0 ) { cs_log ( " ERRORE: Impossibile chiudere PMT fd (errno = % d  % s ) " , errno, strerror (errno)); }
					continua ;
				}

				se (( time_t ) pmt_info. st_mtime ! = demux [i]. pmt_time )
				{
					dvbapi_stop_descrambling (i);
				}

				int32_t ret = close (pmt_fd);
				se (ret < 0 ) { cs_log ( " ERRORE: Impossibile chiudere PMT fd (errno = % d  % s ) " , errno, strerror (errno)); }
				continua ;
			}
			altro
			{
				cs_log ( " Demuxer % d Impossibile aprire il file di PMT % s -> arresto decodifica! " , i, dest);
				dvbapi_stop_descrambling (i);
			}
		}
	}

	se (disable_pmt_files)
	{
		pthread_mutex_unlock (& event_handler_lock);
		ritorno ;
	}

	DIRP = opendir (TMPDIR);
	se (DIRP!)
	{
		cs_log_dbg (D_DVBAPI, " opendir fallito (errore = % d  % s ) " , errno, strerror (errno));
		pthread_mutex_unlock (& event_handler_lock);
		ritorno ;
	}

	mentre (! cs_readdir_r (DIRP, e voce, e dp))
	{
		se (dp) { rompere ; }

		se ( strlen (DP> d_name) < 7 )
			{ continua ; }
		se ( strncmp (DP> d_name, " PMT " , 3 ) =! 0 || strncmp (DP> d_name + strlen (DP> d_name) - 4 , " tmp " , 4 !) = 0 )
			{ continua ; }
# ifdef WITH_STAPI
		struct s_dvbapi_priority p *;
		per (p = dvbapi_priority; p =! NULL ; p = p-> successiva)   // Stapi: verificare se vi è un dispositivo collegato a questo file PMT!
		{
			se (p-> type =! ' s ' ) { continuano ; }   // regola Stapi?
			se ( strcmp (DP> d_name, p-> pmtfile) =! 0 ) { continua ; }   // stesso file?
			rompere ; // risultato trovato!
		}
		se (p == NULL )
		{
			cs_log_dbg (D_DVBAPI, " No corrispondente S: la linea in oscam.dvbapi per pmtfile % s ! -> saltare " , DP-> d_name);
			continua ;
		}
# endif
		snprintf (dest, sizeof (dest), " % s% s " , TMPDIR, DP-> d_name);
		pmt_fd = aperto (dest, O_RDONLY);
		se (pmt_fd < 0 )
			{ continua ; }

		se ( fstat (pmt_fd, e pmt_info)! = 0 )
		{
			int32_t ret = close (pmt_fd);
			se (ret < 0 ) { cs_log ( " ERRORE: Impossibile chiudere PMT fd (errno = % d  % s ) " , errno, strerror (errno)); }
			continua ;
		}

		int32_t trovato = 0 ;
		per (i = 0 ; i <MAX_DEMUX; i ++)
		{
			se ( strcmp (demux [i]. pmt_file , DP-> d_name) == 0 )
			{
				se (( time_t ) pmt_info. st_mtime == demux [i]. pmt_time )
				{
					trovato = 1 ;
					continua ;
				}
				dvbapi_stop_descrambling (i);
			}
		}
		se (trovato)
		{
			int32_t ret = close (pmt_fd);
			se (ret < 0 ) { cs_log ( " ERRORE: Impossibile chiudere PMT fd (errno = % d  % s ) " , errno, strerror (errno)); }
			continua ;
		}

		cs_log_dbg (D_DVBAPI, " ha trovato pmt il file % s " , dest);
		cs_sleepms ( 100 );

		uint32_t len = leggere (pmt_fd, mbuf, sizeof (mbuf));
		int32_t ret = close (pmt_fd);
		se (ret < 0 ) { cs_log ( " ERRORE: Impossibile chiudere PMT fd (errno = % d  % s ) " , errno, strerror (errno)); }

		se (len < 1 )
		{
			cs_log_dbg (D_DVBAPI, " file di pmt % s ha len non valida! " , dest);
			continua ;
		}

		int32_t pmt_id;

# ifdef QBoxHD
		uint32_t  j1 , j2;
		// QboxHD pmt.tmp è la piena capmt scritto come una stringa di valori esadecimali
		// Pmt.tmp deve superare i 3 byte (chars 6 hex) e anche la lunghezza
		se ((len < 6 ) || ((len% 2 )! = 0 ) || ((len / 2 )> sizeof (dest)))
		{
			cs_log_dbg (D_DVBAPI, " errore di analisi QboxHD pmt.tmp, lunghezza non corretta " );
			continua ;
		}

		per (j2 = 0 , J1 = 0 ; j2 <len; j2 + = 2 , J1 ++)
		{
			unsigned  int tmp;
			se ( sscanf (( char *) mbuf + J2 " % 02X " , e tmp)! = 1 )
			{
				cs_log_dbg (D_DVBAPI, " errore di analisi QboxHD pmt.tmp, i dati non validi in posizione % d " , J2);
				pthread_mutex_unlock (& event_handler_lock);
				ritorno ;
			}
			altro
			{
				memcpy (dest + j1 , e tmp, 4 );
			}
		}

		cs_log_dump_dbg (D_DVBAPI, ( unsigned  char *) dest, len / 2 , " QboxHD pmt.tmp: " );
		pmt_id = dvbapi_parse_capmt (( unsigned  char *) dest + 4 , (len / 2 ) - 4 , - 1 , DP-> d_name);
# altro
		se (len> sizeof (dest))
		{
			cs_log_dbg (D_DVBAPI, " tampone event_handler () dest è di piccole dimensioni per i dati di TMP! " );
			continua ;
		}
		se (len < 16 )
		{
			cs_log_dbg (D_DVBAPI, " event_handler () ha ricevuto pmt è troppo piccolo (! % d <16 byte)! " , len);
			continua ;
		}
		cs_log_dump_dbg (D_DVBAPI, mbuf, len, " PMT: " );

		dest [ 0 ] = 0x03;
		dest [ 1 ] = mbuf [ 3 ];
		dest [ 2 ] = mbuf [ 4 ];
		uint32_t pmt_program_length = B2i ( 2 , mbuf + 10 ) e 0xFFF;
		i2b_buf ( 2 , pmt_program_length + 1 , (uchar *) dest + 4 );
		dest [ 6 ] = 0 ;

		memcpy (dest + 7 , mbuf + 12 , len - 12 - 4 );

		pmt_id = dvbapi_parse_capmt ((uchar *) dest, 7 + len - 12 - 4 , - 1 , DP-> d_name);
# endif

		se (pmt_id> = 0 )
		{
			cs_strncpy (demux [pmt_id]. pmt_file , DP> d_name, sizeof (demux [pmt_id]. pmt_file ));
			. demux [pmt_id] pmt_time = ( time_t .) pmt_info st_mtime ;
		}

		se (cfg. dvbapi_pmtmode == 3 )
		{
			disable_pmt_files = 1 ;
			rompere ;
		}
	}
	closedir (DIRP);
	pthread_mutex_unlock (& event_handler_lock);
}

vuoto * dvbapi_event_thread ( vuoto cli *)
{
	struct s_client * client = ( struct s_client *) cli;
	pthread_setspecific (getClient, cliente);
	set_thread_name (__func__);
	mentre ( 1 )
	{
		cs_sleepms ( 750 );
		event_handler ( 0 );
	}

	ritorno  NULL ;
}

vuoto  dvbapi_process_input ( int32_t demux_id, int32_t filter_num, uchar * buffer, int32_t len)
{
	struct s_ecmpids * curpid = & demux [demux_id]. ECMpids [demux [demux_id]. demux_fd [filter_num]. pidindex ];
	int32_t . pid = demux [demux_id] demux_fd [filter_num]. pidindex ; // DeepThought: pid potrebbe essere -1
	uint32_t chid = 0x10000;
	uint32_t ecmlen = ( B2i ( 2 , del buffer + 1 ) e 0xFFF) + 3 ;

	se (demux [demux_id]. demux_fd [filter_num]. Tipo == TYPE_ECM)
	{
		se (len! = 0 )   // len = 0 ricevitore rilevato un bufferoverflow interna!
		{
			cs_log_dbg (D_DVBAPI, " Demuxer % d Filtrare % d dati ECM recuperati (ecmlength = % 03X ) " , demux_id, filter_num + 1 , ecmlen);
			se (( uint ) len <ecmlen) // lunghezza CAT non valida
			{
				cs_log_dbg (D_DVBAPI, " i dati ricevuti con lunghezza totale % 03X ma ​​lunghezza ECM è % 03X -> non valido lunghezza CAT! " , len, ecmlen);
				ritorno ;
			}

			se (! (tampone [ 0 ] == 0x80 || tampone [ 0 ] == 0x81))
			{
				cs_log_dbg (D_DVBAPI, " ha ricevuto un ECM con non valido ecmtable ID % 02X -> ignorando! " , tampone [ 0 ]);
				ritorno ;
			}

			se (curpid-> Tavolo == tampone [ 0 ] &&! caid_is_irdeto (curpid-> CAID))   // attendere pari / dispari cambio ecm (solo non per irdeto!)
				{ ritorno ; }

			se ( caid_is_irdeto (curpid-> CAID))
			{
				// 80 70 39 53 04 05 00 88
				// 81 70 41 41 01 06 00 13 00 06 80 38 52 93 1F D2
				// If (tampone [5]> 20) return;
				se (curpid-> irdeto_maxindex! = tampone [ 5 ])     // 6, registro indice max irdeto
				{
					cs_log_dbg (D_DVBAPI, " Trovato % d Irdeto ECM CHIDs " , tampone [ 5 ] + 1 );
					curpid-> irdeto_maxindex = tampone [ 5 ]; // numchids = 7 (0-6)
				}
			}
		}
		ECM_REQUEST * ER;
		se ((er =! get_ecmtask {())) di ritorno ; }

		ER-> srvid = demux [demux_id]. program_number ;

		ER-> tsid = demux [demux_id]. TSID ;
		ER-> ONID = demux [demux_id]. ONID ;
		ER-> pmtpid = demux [demux_id]. pmtpid ;
		ER-> ens = demux [demux_id]. enigma_namespace ;

		ER-> caid = curpid-> CAID;
		ER-> pid = curpid-> ECM_PID;
		ER-> prid = curpid-> PROVID;
		ER-> vpid = curpid-> VPID;
		ER-> ecmlen = ecmlen;
		memcpy (er-> ecm, tampone, ehm> ecmlen);

		chid = get_subid (er); // recuperare chid o falso chid
		ER-> chid = chid;
		
		se (== len 0 ) // utilizzato solo su ricevitore bufferoverflow interno per ottenere ecm rapidamente fresco FilterData altrimenti congelamento!
		{
			curpid-> Tavolo = 0 ;
			dvbapi_set_section_filter (demux_id, er);
			NULLFREE (er);
			ritorno ;
		}

		se ( caid_is_irdeto (curpid-> CAID))
		{

			se (curpid-> irdeto_curindex! = tampone [ 4 ])    // vecchio stile indice irdeto sbagliato
			{
				se (curpid-> irdeto_curindex == 0xFE)   // controllare se questo ecmfilter appena avviato
				{
					curpid-> irdeto_curindex = Buffer [ 4 ]; // all'avvio impostare l'indice corrente all'indice irdeto del ecm
				}
				altrimenti    // siamo già in esecuzione e non interessato a questo ecm
				{
					se (curpid-> table = tampone [! 0 ]) curpid-> table = 0 ; // correzione per i ricevitori che non supportano sezione filtrante
					dvbapi_set_section_filter (demux_id, er); // impostato filtro ecm per dispari + anche da questa partita doesnt ecm con indice irdeto corrente
					NULLFREE (er);
					ritorno ;
				}
			}
			altro  // fissare per i ricevitori che non supportano sezione filtrante
			{
				se (curpid-> Tavolo == tampone [ 0 ]) {
					NULLFREE (er);
					ritorno ;
				}
			}
			cs_log_dbg (D_DVBAPI, " Demuxer % d ECMTYPE % 02X CAID % 04X PROVID % 06X ECMPID % 04X IRDETO INDEX % 02X MAX INDEX % 02X CHID % 04X CICLO % 02X VPID % 04X " , demux_id, ehm> ecm [ 0 ], ehm -> caid, er-> prid, er-> pid, ehm> ecm [ 4 ], ehm> ecm [ 5 ], er-> chid, curpid-> irdeto_cycle, ehm> vpid);
		}
		altro
		{
			cs_log_dbg (D_DVBAPI, " Demuxer % d ECMTYPE % 02X CAID % 04X PROVID % 06X ECMPID % 04X FAKECHID % 04X (parte unica in ecm) " ,
						  demux_id, ehm> ecm [ 0 ], er-> caid, er-> prid, er-> pid, ehm> chid);
		}

		// Verifica per corrispondenza chid (parte unico ecm in caso di cas non Irdeto) + fix aggiunto per seca2 cambio fakechid mensile 
		se ((curpid-> CHID <0x10000) &&! ((chid == curpid-> CHID) || ((curpid-> CAID >> 8 == 0x01) && (chid & 0xF0FF) == (curpid-> CHID & 0xF0FF)) ))  
		{
			se ( caid_is_irdeto (curpid-> CAID))
			{

				se ((curpid-> irdeto_cycle <0xFE) && curpid-> irdeto_cycle == tampone [ 4 ])    // se stesso: abbiamo pedalato tutti gli indici, ma senza fortuna!
				{
					struct s_dvbapi_priority * forceentry = dvbapi_check_prio_match (demux_id, pid, ' p ' );
					se (! forceentry ||! forceentry-> forza)    // forzata pid? continuare a provare il ecmpid forzato, nessuna forza uccidere filtro ecm
					{
						se (curpid-> controllato == 2 ) {curpid-> verificato = 4 ; }
						se (curpid-> == controllato 1 )
						{
							curpid-> controllato = 2 ;
							curpid-> CHID = 0x10000;
						}
						dvbapi_stop_filternum (demux_id, filter_num); // fermare questo filtro ecm!
						NULLFREE (er);
						ritorno ;
					}
				}
				se (curpid-> irdeto_cycle == 0xFE) {curpid-> irdeto_cycle = tampone [ 4 ]; }   // register indice irdeto di ecm corrente

				curpid-> irdeto_curindex ++; controllo // set sul successivo indice
				se (curpid-> irdeto_curindex> curpid-> irdeto_maxindex) {curpid-> irdeto_curindex = 0 ; }   // controllare se abbiamo raggiunto indice max Irdeto, in tal caso resettare a 0

				curpid-> Tavolo = 0 ;
				dvbapi_set_section_filter (demux_id, er); // impostato filtro ecm per dispari + anche da questa partita doesnt ecm con indice irdeto corrente
				NULLFREE (er);
				ritorno ;
			}
			else   // tutti i sistemi nonirdeto cas
			{
				struct s_dvbapi_priority * forceentry = dvbapi_check_prio_match (demux_id, pid, ' p ' );
				curpid-> Tavolo = 0 ;
				dvbapi_set_section_filter (demux_id, er); // impostato filtro ecm per dispari + anche da questa partita doesnt ecm con indice irdeto corrente
				se (forceentry && forceentry-> forza)
				{
					NULLFREE (er);
					ritorno ; // costretto pid? continuare a provare il ecmpid forzata!
				}
				se (curpid-> controllato == 2 ) {curpid-> verificato = 4 ; }
				se (curpid-> == controllato 1 )
				{
					curpid-> controllato = 2 ;
					curpid-> CHID = 0x10000;
				}
				dvbapi_stop_filternum (demux_id, filter_num); // fermare questo filtro ecm!
				NULLFREE (er);
				ritorno ;
			}
		}

		struct s_dvbapi_priority p *;

		per (p = dvbapi_priority; p =! NULL ; p = p-> successiva)
		{
			se (p-> digitare! = ' l '
					|| (P-> caid && p-> caid! = Curpid-> CAID)
					|| (P-> provid && p-> provid! = Curpid-> PROVID)
					|| (P-> ecmpid && p-> ecmpid! = Curpid-> ECM_PID)
					|| (P-> srvid && p-> srvid! = Demux [demux_id]. program_number ))
				{ continua ; }

			se (( uint ) p-> ritardo == ecmlen && p-> forza < 6 )
			{
				p-> forza ++;
				NULLFREE (er);
				ritorno ;
			}
			se (p-> vigore> = 6 )
				{P-> forza = 0 ; }
		}

		se (! curpid-> PROVID)
			{Curpid-> PROVID = chk_provid (buffer, curpid-> CAID); }

		se ( caid_is_irdeto (curpid-> CAID))    // irdeto: attendere l'indice corretto
		{
			se (tampone [ 4 ]! = curpid-> irdeto_curindex)
			{
				curpid-> Tavolo = 0 ;
				dvbapi_set_section_filter (demux_id, er); // impostato filtro ecm per dispari + anche da questa partita doesnt ecm con indice irdeto corrente
				NULLFREE (er);
				ritorno ;
			}
		}
		// Abbiamo un ecm con l'indice irdeto corretto (o fakechid)
		per (p = dvbapi_priority;! p = null ; p = p-> successiva)   // verificare la presenza di ignorare!
		{
			se ((p-> digitare! = ' i ' )
					|| (P-> caid && p-> caid! = Curpid-> CAID)
					|| (P-> provid && p-> provid! = Curpid-> PROVID)
					|| (P-> ecmpid && p-> ecmpid! = Curpid-> ECM_PID)
					|| (P-> PIDX && p-> pidx- 1 ! = pid)
					|| (P-> srvid && p-> srvid! = Demux [demux_id]. program_number ))
				{ continua ; }

			se (p-> tipo == ' i ' && (p-> chid <0x10000 && p-> chid == chid))     // trovata una corrispondenza chid ignorare con ecm attuale -> ignorando l'indice irdeto
			{
				curpid-> irdeto_curindex ++;
				se (curpid-> irdeto_curindex> curpid-> irdeto_maxindex)     // controllare se curindex è sopra il max
				{
					curpid-> irdeto_curindex = 0 ;
				}
				curpid-> Tavolo = 0 ;
				se ( caid_is_irdeto (curpid-> CAID))    // irdeto: attendere l'indice corretto
				{
					dvbapi_set_section_filter (demux_id, er); // imposta filtro ecm per dispari + anche da questo chid deve essere ignorato!
				}
				altro  // questo fakechid deve essere ignorato, uccidere questo filtro!
				{
					se (curpid-> controllato == 2 ) {curpid-> verificato = 4 ; }
					se (curpid-> == controllato 1 )
					{
						curpid-> controllato = 2 ;
						curpid-> CHID = 0x10000;
					}
					dvbapi_stop_filternum (demux_id, filter_num); // fermare questo filtro ecm!
				}
				NULLFREE (er);
				ritorno ;
			}
		}
		se (er) {
			curpid-> table = er-> ecm [ 0 ];
		}
		request_cw (dvbapi_client, ehm, demux_id, 1 ); // registrare questo ecm per il check risposta ecm ritardata
		ritorno ; // fine ecm filterhandling!
	}

	se (demux [demux_id]. demux_fd [filter_num]. Tipo == TYPE_EMM)
	{
		se (demux [demux_id]. demux_fd [filter_num]. pid == 0x01) // CAT
		{
			cs_log_dbg (D_DVBAPI, " ricevendo gatto " );
			dvbapi_parse_cat (demux_id, tampone, len);

			dvbapi_stop_filternum (demux_id, filter_num);
			ritorno ;
		}
		dvbapi_process_emm (demux_id, filter_num, tampone, len);
	}

	// Filtro emm iterazione
	se (ll_emm_active_filter!)
		{Ll_emm_active_filter = ll_create ( " ll_emm_active_filter " ); }

	se (ll_emm_inactive_filter!)
		{Ll_emm_inactive_filter = ll_create ( " ll_emm_inactive_filter " ); }

	se (ll_emm_pending_filter!)
		{Ll_emm_pending_filter = ll_create ( " ll_emm_pending_filter " ); }

	uint32_t filter_count = ll_count (ll_emm_active_filter) + ll_count (ll_emm_inactive_filter);

	se (demux [demux_id]. max_emm_filter > 0
			&& ll_count (ll_emm_inactive_filter)> 0
			&& Filter_count> demux [demux_id]. max_emm_filter )
	{

		int32_t filter_queue = ll_count (ll_emm_inactive_filter);
		int32_t fermò = 0 , ha iniziato = 0 ;
		struct voltaB ora;
		cs_ftime (e ora);

		struct s_emm_filter * filter_item;
		LL_ITER itr;
		itr = ll_iter_create (ll_emm_active_filter);

		mentre ((filter_item = ll_iter_next (& ITR))! = NULL )
		{
			se (! ll_count (ll_emm_inactive_filter) || iniziato filter_queue ==)
				{ rompere ; }
			int64_t andato = comp_timeb (e ora, e filter_item-> time_started);
			se (andata> 45 * 1000 )
			{
				struct s_dvbapi_priority * forceentry = dvbapi_check_prio_match_emmpid (filter_item-> demux_id, filter_item-> caid,
													   filter_item-> provid, ' p ' );

				se (! forceentry || (forceentry &&! forceentry-> forza))
				{
					cs_log_dbg (D_DVBAPI, " Rimozione filtro emm % d sull'indice demux % d " , filter_item-> num, filter_item-> demux_id);
					dvbapi_stop_filternum (filter_item-> demux_id, filter_item-> num - 1 );
					ll_iter_remove_data (& ITR);
					add_emmfilter_to_list (filter_item-> demux_id, filter_item-> Filtro, filter_item-> caid,
										  filter_item-> provid, filter_item-> pid, - 1 , falso );
					fermato ++;
				}
			}

			int32_t ret;
			se (arrestato> avviato)
			{
				struct s_emm_filter * filter_item2;
				LL_ITER itr2 = ll_iter_create (ll_emm_inactive_filter);

				mentre ((filter_item2 = ll_iter_next (e itr2)))
				{
					cs_log_dump_dbg (D_DVBAPI, filter_item2-> Filtro, 32 , " Avvio di filtro emm pid: 0x ​​% 04X sull'indice demux % i " , filter_item2-> pid, filter_item2-> demux_id);
					ret = dvbapi_set_filter (filter_item2-> demux_id, selected_api, filter_item2-> pid, filter_item2-> caid,
											filter_item2-> provid, filter_item2-> Filtro, filter_item2-> Filtro + 16 , 0 ,
											demux [filter_item2-> demux_id]. pidindex , TYPE_EMM, 1 );
					se (ret = - 1 )
					{
						ll_iter_remove_data (& itr2);
						iniziato ++;
						rompere ;
					}
				}
			}
		}

		itr = ll_iter_create (ll_emm_pending_filter);

		mentre ((filter_item = ll_iter_next (& ITR))! = NULL )
		{
			add_emmfilter_to_list (filter_item-> demux_id, filter_item-> Filtro, filter_item-> caid, filter_item-> provid, filter_item-> pid, 0 , falso );
			ll_iter_remove_data (& ITR);
		}
	}
}

static  vuoto * dvbapi_main_local ( vuoto * cli)
{
	int32_t i, j;
	struct s_client * client = ( struct s_client *) cli;
	client-> thread = pthread_self ();
	pthread_setspecific (getClient, cli);

	dvbapi_client = cli;

	int32_t maxpfdsize = (MAX_DEMUX * maxfilter) + MAX_DEMUX + 2 ;
	struct pollfd pfd2 [maxpfdsize];
	struct inizio voltaB, fine;   // ora di inizio polling, ora di fine sondaggio
# definire  PMT_SERVER_SOCKET  " /tmp/.listen.camd.socket "
	struct sockaddr_un saddr;
	saddr. sun_family = AF_UNIX;
	strncpy (. saddr sun_path , PMT_SERVER_SOCKET, 107 );
	. saddr sun_path [ 107 ] = ' \ 0 ' ;

	int32_t rc, pfdcount, g, connfd, clilen;
	int32_t ids [maxpfdsize], FDN [maxpfdsize], digitare [maxpfdsize];
	struct SOCKADDR servaddr;
	ssize_t len = 0 ;
	uchar mbuf [ 1024 ];

	struct s_auth * conto;
	int32_t ok = 0 ;
	per (conto = CFG. conto ;! account = null ; conto = contabile> successiva)
	{
		se ((ok = is_dvbapi_usr (contabilità> usr)))
			{ rompere ; }
	}
	cs_auth_client (? cliente, ok conto: ( struct s_auth *) (- 1 ), " dvbapi " );

	memset (demux, 0 , sizeof ( struct demux_s) * MAX_DEMUX);
	memset (ca_fd, 0 , sizeof (ca_fd));
	memset (unassoc_fd, 0 , sizeof (unassoc_fd));

	dvbapi_read_priority ();
	dvbapi_load_channel_cache ();
	dvbapi_detect_api ();

	se (selected_box == - 1 || selected_api == - 1 )
	{
		cs_log ( " ERRORE: Non potrebbe rilevare la versione dvbapi. " );
		ritorno  NULL ;
	}

	se (cfg. dvbapi_pmtmode == 1 )
		{disable_pmt_files = 1 ; }

	int32_t listenfd = - 1 ;
	se (cfg. dvbapi_boxtype ! = BOXTYPE_IPBOX_PMT && cfg. dvbapi_pmtmode ! = 2 && cfg. dvbapi_pmtmode ! = 5 && cfg. dvbapi_pmtmode ! = 6 )
	{
		se (! CFG. dvbapi_listenport )
			listenfd = dvbapi_init_listenfd ();
		altro
			listenfd = dvbapi_net_init_listenfd ();
		se (listenfd < 1 )
		{
			cs_log ( " ERRORE: Impossibile socket non init: (errno = % d : % s ) " , errno, strerror (errno));
			ritorno  NULL ;
		}
	}

	pthread_mutex_init (& event_handler_lock, NULL );

	per (i = 0 ; i <MAX_DEMUX; i ++)   // init tutti demuxer!
	{
		demux [i]. pidindex = - 1 ;
		demux [i]. curindex = - 1 ;
	}

	se (cfg. dvbapi_pmtmode ! = 4 && cfg. dvbapi_pmtmode ! = 5 && cfg. dvbapi_pmtmode ! = 6 )
	{
		struct  sigaction signal_action;
		signal_action. sa_handler = event_handler;
		sigemptyset (. & signal_action sa_mask );
		signal_action. sa_flags = SA_RESTART;
		sigaction (SIGRTMIN + 1 , e signal_action, NULL );

		dir_fd = aperto (TMPDIR, O_RDONLY);
		se (dir_fd> = 0 )
		{
			fcntl (dir_fd, F_SETSIG, SIGRTMIN + 1 );
			fcntl (dir_fd, F_NOTIFY, DN_MODIFY | DN_CREATE | DN_DELETE | DN_MULTISHOT);
			event_handler (SIGRTMIN + 1 );
		}
	}
	altro
	{
		pthread_t event_thread;
		int32_t ret = pthread_create (& event_thread, NULL , dvbapi_event_thread, ( vuoto *) dvbapi_client);
		se (in pensione)
		{
			cs_log ( " ERRORE: Impossibile creare dvbapi filo evento (errno = % d  % s ) " , ret, strerror (in pensione));
			ritorno  NULL ;
		}
		altro
			{ pthread_detach (event_thread); }
	}

	se (listenfd = - 1 )
	{
		pfd2 [ 0 ]. fd = listenfd;
		pfd2 [ 0 ]. eventi = (POLLIN | POLLPRI);
		digitare [ 0 ] = 1 ;
	}

# ifdef WITH_COOLAPI
	sistema ( " pzapit -rz " );
# endif
	cs_ftime (& start); // registrati ora di inizio
	mentre ( 1 )
	{
		se (pausecam)   // per dbox2, STAPI o PC in modalità standby dont analizzare qualsiasi ecm / emm o cercare di iniziare il prossimo filtro
			{ continua ; }

		se (cfg. dvbapi_pmtmode == 6 )
		{
			se (listenfd < 0 )
			{
				cs_log ( " PMT6: Tentativo connettersi a enigma CA PMT ascoltare presa ... " );
				/ * Presa init * /
				se ((listenfd = presa (AF_UNIX, SOCK_STREAM, 0 )) < 0 )
				{
					
					cs_log ( " Errore di presa (errno = % d  % s ) " , errno, strerror (errno));
					listenfd = - 1 ;
				}
				altrimenti  se ( collegamento (listenfd, ( struct sockaddr *) e saddr, sizeof (saddr)) < 0 )
				{
					cs_log ( " presa collegare errore (errore = % d  % s ) " , errno, strerror (errno));
					vicino (listenfd);
					listenfd = - 1 ;
				}
				altro
				{
					pfd2 [ 0 ]. fd = listenfd;
					pfd2 [ 0 ]. eventi = (POLLIN | POLLPRI);
					digitare [ 0 ] = 1 ;
					cs_log ( " PMT6 CA PMT Server collegato su fd % d ! " , listenfd);
				}
			}

		}
		pfdcount = (listenfd> - 1 )? 1 : 0 ;

		per (i = 0 ; i <MAX_DEMUX; i ++)
		{	
			// Aggiunge fd client di che non sono ancora associati al demux ma ha bisogno di essere intervistati per i dati
			se (unassoc_fd [i]) {
				pfd2 [pfdcount]. fd = unassoc_fd [i];
				pfd2 [pfdcount]. eventi = (POLLIN | POLLPRI);
				digitare [pfdcount ++] = 1 ;
			}

			se (demux [i]. program_number == 0 ) { continua ; }   // solo evalutate demuxer che hanno canali assegnati
			
			uint32_t ecmcounter = 0 , emmcounter = 0 ;
			per (g = 0 ; g <maxfilter; g ++)
			{
				se (demux [i]. demux_fd [g]. fd <= 0 ) continuare ; // negano ovvio fd non valido!
				
				se (! cfg. dvbapi_listenport && cfg. dvbapi_boxtype ! = BOXTYPE_PC_NODMX && selected_api! = STAPI && selected_api! = COOLAPI)
				{
					. pfd2 [pfdcount] fd = demux [i]. demux_fd [g]. fd ;
					pfd2 [pfdcount]. eventi = (POLLIN | POLLPRI);
					ids [pfdcount] = i;
					FDN [pfdcount] = g;
					digitare [pfdcount ++] = 0 ;
				}
				se (. demux [i] demux_fd [g]. Tipo == TYPE_ECM) {ecmcounter ++; }   // contare filtri ECM per vedere se è possibile comunque demuxing
				se (. demux [i] demux_fd [g]. Tipo == TYPE_EMM) {emmcounter ++; }   // conteggio emm filtri anche
			}
			se (ecmcounter! = demux [i]. old_ecmfiltercount || emmcounter! = demux [i]. old_emmfiltercount )    // solo producono il login se qualcosa è cambiato
			{
				cs_log_dbg (D_DVBAPI, " Demuxer % d ha % d ecmpids, % d streampids, % d ecmfilters e % d di max % d emmfilters " , i, demux [i]. ECMpidcount ,
							  . demux [i] STREAMpidcount - 1 ., ecmcounter, emmcounter, demux [i] max_emm_filter );
				demux [i]. old_ecmfiltercount = ecmcounter; // salvare il nuovo importo di ecmfilters
				demux [i]. old_emmfiltercount = emmcounter; // salvare il nuovo importo di emmfilters
			}

			// Inizio emm ritardata per caids non Irdeto, iniziare emm gatto se non già fatto per questa demuxer!
			
			struct voltaB ora;
			cs_ftime (e ora);
			
			se (. CFG dvbapi_au > 0 . && demux [i] emm_filter == - 1 . && demux [i] EMMpidcount == 0 && emmcounter == 0 )
			{
				int64_t andato = comp_timeb (e ora, e demux [i]. emmstart );
				se (andata> 30 * 1000 ) {
					cs_ftime (e demux [i]. emmstart ); // trucco per lasciare inizio emm recupero dopo 30 secondi per accelerare zapping
					dvbapi_start_filter (i, demux [i]. pidindex , 0x001, 0x001, 0x01, 0x01, 0xFF, 0 , TYPE_EMM); // CAT
				}
				// Continua; // Procedere con la prossima demuxer
			}

			// Inizio per irdeto in quanto hanno bisogno emm prima ecm (PMT emmstart = 1 se rilevato 0x06 caid)
			int32_t emmstarted = demux [i]. emm_filter ;
			se (cfg. dvbapi_au && demux [i]. EMMpidcount > 0 )    // controllare ogni volta da quando i lettori di condivisione ci potrebbe dare nuovi filtri dovuti al cambiamento hexserial
			{
				se (emmcounter && == emmstarted - 1 )
				{
					. demux [i] emmstart = ora;
					dvbapi_start_emm_filter (i); // avviare emmfiltering se vengono trovati emmpids
				}
				altro
				{
					int64_t andato = comp_timeb (e ora, e demux [i]. emmstart );
					se (andata> 30 * 1000 )
					{
						. demux [i] emmstart = ora;
						dvbapi_start_emm_filter (i); // avviare emmfiltering ritardato se i filtri già correvano
					}
				}
				// If (! = Demux emmstarted [i] .emm_filter && emmcounter) {continua; } // Procedere con la prossima demuxer se non emms cui esecuzione prima
			}

			se (ecmcounter == 0 && demux [i]. ECMpidcount > 0 )    // Riavviare decodifica tutti caids abbiamo ecmpids ma senza filtri ECM!
			{

				int32_t iniziato = 0 ;

				per (g = 0 ; g <demux [i]. ECMpidcount ; g ++)   // evitare gara: non tutti i pid viene chiesto e controllato ancora!
				{
					se (demux [i]. ECMpids [g]. controllato == 0 && demux [i]. ECMpids [g]. stato > = 0 )   // controllare se run prio è fatto
					{
						dvbapi_try_next_caid (i, 0 ); // non si fa, in modo da iniziare il prossimo prio pid
						iniziato = 1 ;
						rompere ;
					}
				}
				se (iniziato) { continua ; }   // se iniziata un filtro procedere con la prossima demuxer

				se (g == demux [i]. ECMpidcount )    // tutti pid utilizzabili (con prio) sono provati, lascia ricominciare da capo senza prio!
				{
					per (g = 0 ; g <demux [i]. ECMpidcount ; g ++)   // evitare gara: non tutti i pid viene chiesto e controllato ancora!
					{
						se (demux [i]. ECMpids [g]. controllato == 2 && demux [i]. ECMpids [g]. stato > = 0 )   // controlla se noprio corsa è fatto
						{
							demux [i]. ECMpids [g]. irdeto_curindex = 0xFE;
							. demux [i] ECMpids [g]. irdeto_maxindex = 0 ;
							demux [i]. ECMpids [g]. irdeto_cycle = 0xFE;
							. demux [i] ECMpids . [g] tenta = 0xFE;
							. demux [i] ECMpids . [g] tabella = 0 ;
							. demux [i] ECMpids . [g] CHID = 0x10000; // rimuovere chid prio
							dvbapi_try_next_caid (i, 2 ); // non si fa, in modo da iniziare la prossima no prio pid
							iniziato = 1 ;
							rompere ;
						}
					}
				}
				se (iniziato) { continua ; }   // se iniziata un filtro procedere con la prossima demuxer

				se (g == demux [i]. ECMpidcount )    // tutti pid utilizzabili sono provati, consente di iniziare da capo!
				{
					se (demux [i]. decodingtries == - 1 ) // prima redecoding tentativo?
					{
						cs_ftime (e demux [i]. decstart );
						per (g = 0 ; g <demux [i]. ECMpidcount ; g ++)   // reinit alcune cose usate da seconda corsa (senza prio)
						{
							demux [i]. ECMpids [g]. controllato = 0 ;
							demux [i]. ECMpids [g]. irdeto_curindex = 0xFE;
							. demux [i] ECMpids [g]. irdeto_maxindex = 0 ;
							demux [i]. ECMpids [g]. irdeto_cycle = 0xFE;
							. demux [i] ECMpids . [g] tabella = 0 ;
							demux [i]. decodingtries = 0 ;
							dvbapi_edit_channel_cache (i, g, 0 ); // rimuovere questo pid da channelcache da quando abbiamo avuto fonda su qualsiasi ecmpid!
						}
					}
					uint8_t number_of_enabled_pids = 0 ;
					. demux [i] decodingtries ++;
					dvbapi_resort_ecmpids (I);
					
					per (g = 0 ; g <demux [i]. ECMpidcount ; g ++)   // contare il numero di pid abilitati!
					{
						se (. demux [i] ECMpids . [g] stato > = 0 ) number_of_enabled_pids ++;
					}
					se (! number_of_enabled_pids)
					{
						se (demux [i]. decodingtries == 10 )
						{
							demux [i]. decodingtries = 0 ;
							cs_log ( " Demuxer % d non ecmpids corrispondente abilitati -> decodifica è in attesa per i lettori di corrispondenza! " , i);
						}
					}
					altro
					{
						cs_ftime (e demux [i]. decend );
						demux [i]. decodingtries = - 1 ; // Reset per primo run di nuovo!
						int64_t andato = comp_timeb (e demux [i]. decend , e demux [i]. decstart );
						cs_log ( " Demuxer % d riavvio decodingrequests dopo % " PRId64 " ms con % d abilitato e % d ecmpids disabili! " , io, andato, number_of_enabled_pids,
							(. Demux [i] ECMpidcount -number_of_enabled_pids));
						dvbapi_try_next_caid (i, 0 );
					}
				}
			}

			se (demux [i]. socket_fd > 0 && cfg. dvbapi_pmtmode ! = 6 )
			{
				rc = 0 ;
				per (j = 0 ; j <pfdcount; j ++)
				{
					se (pfd2 [j]. fd == demux [i]. socket_fd )
					{
						rc = 1 ;
						rompere ;
					}
				}
				se (rc == 1 ) { continua ; }

				. pfd2 [pfdcount] fd . = demux [i] socket_fd ;
				pfd2 [pfdcount]. eventi = (POLLIN | POLLPRI);
				ids [pfdcount] = i;
				digitare [pfdcount ++] = 1 ;
			}
		}

		mentre ( 1 )
		{
			rc = sondaggio (pfd2, pfdcount, 300 );
			se (listenfd == - 1 && cfg. dvbapi_pmtmode == 6 ) { rompere ; }
			se (rc < 0 )
				{ continua ; }
			rompere ;
		}

		se (rc> 0 )
		{
			cs_ftime (e finale); // registrati ora di fine
			int64_t timeout = comp_timeb (e fine, e iniziare a);
			se (timeout < 0 ) {
				cs_log ( " *** ATTENZIONE: brutto momento di compromettere l'intera OSCAM ECM MANIPOLAZIONE **** " );
			}
			cs_log_dbg (D_TRACE, " Nuovi eventi si è verificato il % d di % d gestori dopo % " PRId64 " ms inattività " , rc, pfdcount, timeout);
			cs_ftime (& start); // registrare nuovo tempo di inizio per il prossimo sondaggio
		}

		per (i = 0 ; i <pfdcount && rc> 0 ; i ++)
		{
			se (pfd2 [i]. revents == 0 ) { continua ; }   // saltare prese senza modifiche
			rc--; // evento gestito!
			cs_log_dbg (D_TRACE, " Ora la gestione fd % d che ha segnalato evento % d " , pfd2 [i]. fd , pfd2 [i]. revents );

			se (pfd2 [i]. revents & (POLLHUP | POLLNVAL | POLLERR))
			{
				se (tipo [i] == 1 )
				{
					per (j = 0 ; j <MAX_DEMUX; j ++)
					{
						se (demux [j]. socket_fd == pfd2 [i]. fd )   // se listenfd chiude fermare tutto decodifica assegnato!
						{
							dvbapi_stop_descrambling (j);
						}
					}
					int32_t ret = close (pfd2 [i]. fd );
					se (ret < 0 && errno =! 9 ) { cs_log ( " Errore: impossibile chiudere presa demuxer fd (errno = % d  % s ) " , errno, strerror (errno)); }
					se (pfd2 [i]. fd == listenfd && cfg. dvbapi_pmtmode == 6 )
					{
						listenfd = - 1 ;
					}
				}
				altro    // type = 0
				{
					int32_t demux_index = id [i];
					int32_t n = FDN [i];
					dvbapi_stop_filternum (demux_index, n); // filtro di arresto in quanto i suoi dà errori e voleva tornare nulla di buono.
				}
				continuare ; // continuare con altri eventi
			}

			se (pfd2 [i]. revents & (POLLIN | POLLPRI))
			{
				se (tipo [i] == 1 && pmthandling == 0 )
				{
					pmthandling = 1 ;      // pmthandling in corso!
					pmt_stopmarking = 0 ; // per stop_descrambling marcatura PMT modalità 6
					connfd = - 1 ;          // inizialmente non presa a leggere
					int add_to_poll = 0 ; // potremmo aver bisogno di interrogare ulteriormente questa presa quando non ci sono dati PMT viene in

					se (pfd2 [i]. fd == listenfd)
					{
						se (cfg. dvbapi_pmtmode == 6 ) {
							connfd = listenfd;
							disable_pmt_files = 1 ;
						} altro {
							clilen = sizeof (servaddr);
							connfd = accettare (listenfd, ( struct sockaddr *) & servaddr, ( socklen_t *) e clilen);
							cs_log_dbg (D_DVBAPI, " nuovo fd connessione socket: % d " , connfd);
							se (cfg. dvbapi_listenport )
							{
								// Aggiornamento WebIf dati
								client-> ip = SIN_GET_ADDR (servaddr);
								client-> port = ntohs ( SIN_GET_PORT (servaddr));
							}
							add_to_poll = 1 ;

							se (cfg. dvbapi_pmtmode == 3 || cfg. dvbapi_pmtmode == 0 {disable_pmt_files =) 1 ; }

							se (connfd <= 0 )
								cs_log_dbg (D_DVBAPI, " accept () restituisce l'errore in caso fd % d (errno = % d  % s ) " , pfd2 [i]. revents , ermo, strerror (errno));
						}
					}
					altro
					{
						cs_log_dbg (D_DVBAPI, " PMT Aggiornamento sulla presa % d . " , pfd2 [i]. fd );
						connfd = pfd2 [i]. fd ;
					}

					// lettura e il completamento dei dati provenienti da presa
					se (connfd> 0 ) {
						uint32_t pmtlen = 0 , chunks_processed = 0 ;

						int prova = 100 ;
						fare {
							len = recv (connfd, mbuf + pmtlen, sizeof (mbuf) - pmtlen, MSG_DONTWAIT);
							se (len> 0 )
								pmtlen + = len;
							se ((CFG. dvbapi_listenport || cfg. dvbapi_boxtype == BOXTYPE_PC_NODMX) &&
								(LEN == 0 || (LEN == - 1 ! && (errno = EINTR && errno = EAGAIN))))
							{
								// Client si disconnette, si fermano tutti decodifica assegnato
								cs_log_dbg (D_DVBAPI, " Socket % d riferito stretto legame " , connfd);
								int active_conn = 0 ; // altro contatore connessioni attive
								per (j = 0 ; j <MAX_DEMUX; j ++)
								{
									se (demux [j]. socket_fd == connfd)
										dvbapi_stop_descrambling (j);
									altrimenti  se (demux [j]. socket_fd )
										active_conn ++;
									// Togliere dal unassoc_fd quando necessario
									se (unassoc_fd [j] == connfd)
										unassoc_fd [j] = 0 ;
								}
								vicino (connfd);
								connfd = - 1 ;
								add_to_poll = 0 ;
								se (! active_conn) // Ultimo collegamento chiuso
								{
									client_proto_version = 0 ;
									se (client_name)
									{
										gratuito (client_name);
										client_name = NULL ;
									}
									se (cfg. dvbapi_listenport )
									{
										// Aggiornamento WebIf dati
										client-> ip = get_null_ip ();
										client-> port = 0 ;
									}
								}
								rompere ;
							}
							se (pmtlen> = 8 ) // se abbiamo ricevuto meno di 8 byte, che non è completa di sicuro
							{
								// Controllare e cercare di elaborare completo oggetti PMT e filtrare i dati da blocchi per evitare PMT buffer overflow
								uint32_t opcode_ptr;
								memcpy (& opcode_ptr, e mbuf [ 0 ], 4 );                      // utilizzati al solo fine del compilatore in silenzio avverte circa dereferenziazione puntatore tipo punned
								uint32_t codice operativo = ntohl (opcode_ptr);                   // ottenere il codice operativo client (4 byte)
								uint32_t chunksize = 0 ;                                // dimensioni di completa chunk nel buffer (un codice operativo con i dati)
								uint32_t data_len = 0 ;                                 // variabile per lunghezza dati interni (es. per la dimensione dei dati del filtro, PMT len)

								// Rilevare il codice operativo, le sue dimensioni (chunksize) e la sua dimensione dei dati interni (data_len)
								se ((codice operativo e 0xFFFFF000) == DVBAPI_AOT_CA)
								{
									// Dimensioni parse pacchetto (ASN.1)
									uint32_t size = 0 ;
									se (mbuf [ 3 ] e 0x80)
									{
										data_len = 0 ;
										size = mbuf [ 3 ] & 0x7F;
										se (pmtlen> 4 + formato)
										{
											uint32_t k;
											per (k = 0 ; k <dimensione; k ++)
												data_len = (data_len << 8 ) | mbuf [ 3 + 1 + k];
											formato ++;
										}
									}
									altro
									{
										data_len = mbuf [ 3 ] & 0x7F;
										size = 1 ;
									}
									chunksize = 3 + size + data_len;
								}
								altro  interruttore (codice operativo)
								{
									caso DVBAPI_FILTER_DATA:
									{
										data_len = B2i ( 2 , mbuf + 7 ) e 0x0fff;
										chunksize = 6 + 3 + data_len;
										rompere ;
									}
									caso DVBAPI_CLIENT_INFO:
									{
										data_len = mbuf [ 6 ];
										chunksize = 6 + 1 + data_len;
										rompere ;
									}
									predefinito :
										cs_log ( " comando presa sconosciuto ricevuto: 0x % 08X " , codice operativo);
								}

								// Elaborare i dati completi in base al tipo
								se (chunksize < sizeof (mbuf) && chunksize <= pmtlen) // gestire solo se abbiamo esagerato un chunksize completo!
								{
									chunks_processed ++;
									se ((codice operativo e 0xFFFFF000) == DVBAPI_AOT_CA)
									{
										cs_log_dump_dbg (D_DVBAPI, mbuf, chunksize, " Analisi % d oggetto PMT (s): " , chunks_processed);
										dvbapi_handlesockmsg (mbuf, chunksize, connfd);
										add_to_poll = 0 ;
										se (cfg. dvbapi_listenport && codice operativo == DVBAPI_AOT_CA_STOP)
											add_to_poll = 1 ;
									}
									altro  interruttore (codice operativo)
									{
										caso DVBAPI_FILTER_DATA:
										{
											int32_t demux_index = mbuf [ 4 ];
											int32_t filter_num = mbuf [ 5 ];
											dvbapi_process_input (demux_index, filter_num, mbuf + 6 , data_len + 3 );
											rompere ;
										}
										caso DVBAPI_CLIENT_INFO:
										{
											uint16_t client_proto_ptr;
											memcpy (& client_proto_ptr, e mbuf [ 4 ], 2 );
											uint16_t client_proto = ntohs (client_proto_ptr);
											se (client_name)
												gratuito (client_name);
											se ( cs_malloc (& client_name, data_len + 1 ))
											{
												memcpy (CLIENT_NAME, e mbuf [ 7 ], data_len);
												client_name [data_len] = 0 ;
												cs_log ( " Cliente collegato: ' % s '(versione di protocollo = % d ) " , client_name, client_proto);
											}
											client_proto_version = client_proto; // impostare la var globale in funzione del cliente

											// Come risposta stiamo inviando il nostro informazioni al cliente:
											dvbapi_net_send (DVBAPI_SERVER_INFO, connfd, - 1 , - 1 , NULL );
											rompere ;
										}
									}

									se (pmtlen == chunksize) // se abbiamo recuperato e gestito il chunksize esatto contatore buffer di reset!
										pmtlen = 0 ;

									// Se leggiamo più dati poi elaborati, spostarlo all'inizio
									se (pmtlen> chunksize)
									{
										memmove (mbuf, mbuf + chunksize, pmtlen - chunksize);
										pmtlen - = chunksize;
									}
									continua ;
								}
							}
							se (len <= 0 ) {
								se (pmtlen> 0 || chunks_processed> 0 ) // tutti i dati letti
									rompere ;
								altrimenti {           // attesa per i dati saranno disponibili e riprovare

									// Rimuovere dal unassoc_fd se il fd presa non è valido
									se (errno == EBADF)
										per (j = 0 ; j <MAX_DEMUX; j ++)
											se (unassoc_fd [j] == connfd)
												unassoc_fd [j] = 0 ;
									cs_sleepms ( 20 );
									continua ;
								}
							}
						} mentre (pmtlen < sizeof (mbuf) && tries--);

						// Se la connessione è nuovo e abbiamo letto non ci sono dati PMT, quindi aggiungerla al sondaggio,
						// Altrimenti questa presa non verrà controllato con sondaggio quando arives dati
						// Perché fd non è ancora assegnato con il demux
						se (add_to_poll) {
							per (j = 0 ; j <MAX_DEMUX; j ++) {
								se (! unassoc_fd [j]) {
									unassoc_fd [j] = connfd;
									rompere ;
								}
							}
						}

						se (pmtlen> 0 ) {
							se (pmtlen < 3 )
								cs_log_dbg (D_DVBAPI, " messaggio di server CA PMT troppo breve! " );
							altro {
								se (pmtlen> = sizeof (mbuf))
									cs_log ( " ***** ATTENZIONE: PMT BUFFER, si prega di segnalare ******! " );
								cs_log_dump_dbg (D_DVBAPI, mbuf, pmtlen, " Nuovo PMT informazioni da presa (dimensione totale: % d ) " , pmtlen);
								dvbapi_handlesockmsg (mbuf, pmtlen, connfd);
							}
						}
					}
					pmthandling = 0 ; // pmthandling fatto!
					continuare ; // continuare con altri eventi!
				}
				altro      // tipo == 0
				{
					int32_t demux_index = id [i];
					int32_t n = FDN [i];

					se ((len = dvbapi_read_device (pfd2 [i]. fd , mbuf, sizeof (mbuf))) <= 0 ) // sempre leggere a vuoto ricevitore databuffer
					{
						se (( int .) demux [demux_index] demux_fd . [n] fd !. = pfd2 [i] fd ) { continua ; } // ma se filtro già ucciso senza bisogno di elaborare i dati!
						
						se (len < 0 ) // grave FilterData errore di lettura
						{
							dvbapi_stop_filternum (demux_index, n); // filtro di arresto in quanto i suoi dà errori e voleva tornare nulla di buono.
							maxfilter--; // maxfilters inferiori per evitare questo con le nuove impostazioni del filtro!
							continua ;
						}
						se (! len) // ricevitore troppopieno filterbuffer interno
						{
							memset (mbuf, 0 , sizeof (mbuf));
						}
					}

					dvbapi_process_input (demux_index, n, mbuf, len);
				}
				continuare ; // continuare con altri eventi!
			}
		}
	}
	ritorno  NULL ;
}

vuoto  dvbapi_write_cw ( int32_t demux_id, uchar * cw, int32_t pid)
{
	int32_t n;
	int8_t cwEmpty = 0 ;
	unsigned  char nullcw [ 8 ];
	memset (nullcw, 0 , 8 );
	ca_descr_t ca_descr;

	memset (& ca_descr, 0 , sizeof (ca_descr));

	se ( memcmp (demux [demux_id]. lastcw [ 0 ], nullcw, 8 ) == 0
			&& memcmp (demux [demux_id]. lastcw [ 1 ], nullcw, 8 ) == 0 )
		{CwEmpty = 1 ; } // per fare in modo che entrambi i cws avere scritto su constantcw


	per (n = 0 ; n < 2 ; n ++)
	{
		char lastcw [ 9 * 3 ];
		char newcw [ 9 * 3 ];
		cs_hexdump ( 0 , demux [demux_id]. lastcw [n], 8 , lastcw, sizeof (lastcw));
		cs_hexdump ( 0 , cw + (n * 8 ), 8 , newcw, sizeof (newcw));

		se (( memcmp (cw + (n * 8 ), demux [demux_id]. lastcw [n], 8 )! = 0 || cwEmpty)
				&& memcmp (cw + (n * 8 ), nullcw, 8 )! = 0 ) // verificare se parte cw già consegnato e nuovo è valido!
		{
			int32_t idx = dvbapi_ca_setpid (demux_id, pid);   // preparare ca
			. ca_descr index = idx;
			. ca_descr parity = n;
			cs_log_dbg (D_DVBAPI, " Demuxer % d scrittura % s parte ( % s ) di controlword, sostituendo scaduto ( % s ) " , demux_id, (n == 1 ? " , anche " : " dispari " ),
						  newcw, lastcw);
			memcpy (. demux [demux_id] lastcw [n], cw + (n * 8 ), 8 );
			memcpy (. ca_descr cw , cw + (n * 8 ), 8 );

# ifdef WITH_COOLAPI
			cs_log_dbg (D_DVBAPI, " Demuxer % d scrittura cw % d Indice: % d (ca_mask % d ) " , demux_id, n, ca_descr. Indice , demux [demux_id]. ca_mask );
			coolapi_write_cw (. demux [demux_id] ca_mask , demux [demux_id]. STREAMpids , demux [demux_id]. STREAMpidcount , e ca_descr);
# altro
			int32_t i;
			per (i = 0 ; i <MAX_DEMUX; i ++)
			{
				se (demux [demux_id]. ca_mask & ( 1 << i))
				{
					cs_log_dbg (D_DVBAPI, " Demuxer % d scrittura cw % d Indice: % d (ca % d ) " , demux_id, n, ca_descr. Indice , i);

					se (cfg. dvbapi_boxtype == BOXTYPE_PC || cfg. dvbapi_boxtype == BOXTYPE_PC_NODMX)
						dvbapi_net_send (. DVBAPI_CA_SET_DESCR, demux [demux_id] socket_fd , demux_id, - 1  / * inutilizzato * / , ( unsigned  char *) e ca_descr);
					altro
					{
						se (ca_fd [i] <= 0 )
						{
							ca_fd [i] = dvbapi_open_device ( 1 , i, demux [demux_id]. adapter_index );
							se (ca_fd [i] <= 0 )
								{ continua ; } // procedere prossimo ruscello
						}
						se ( dvbapi_ioctl (ca_fd [i], CA_SET_DESCR, e ca_descr) < 0 ) {
							cs_log ( " ERROR: ioctl (CA_SET_DESCR): % s " , strerror (errno));
						}
					}
				}
			}
# endif
		}
	}
}

vuoto  delayer (ECM_REQUEST * er)
{
	se (. CFG dvbapi_delayer <= 0 ) { ritorno ; }

	struct voltaB tpe;
	cs_ftime (e TPE);
	int64_t andato = comp_timeb (e TPE, e er-> tps);
	se (andata <cfg. dvbapi_delayer )
	{
		cs_log_dbg (D_DVBAPI, " delayer: andato = % " PRId64 " ms, cfg = % d ms -> ritardo = % " PRId64 " ms " , andato, cfg. dvbapi_delayer , CFG. dvbapi_delayer - gone);
		cs_sleepms (. CFG dvbapi_delayer - gone);
	}
}

vuoto  dvbapi_send_dcw ( struct s_client * cliente, ECM_REQUEST er *)
{
	int32_t i, j, gestito = 0 ;

	per (i = 0 ; i <MAX_DEMUX; i ++)
	{
		uint32_t nocw_write = 0 ; // 0 = cw scrittura, 1 = Dont write cw a demuxer hardware
		se (demux [i]. program_number == 0 ) { continua ; }   // ignorare demuxer vuoti
		se (demux [i]. program_number = er-> srvid!) { continua ; }   // saltare risposta ecm per altri srvid
		. demux [i] rdr = er-> selected_reader;
		per (j = 0 ; j <demux [i]. ECMpidcount ; j ++)   // controllare corrispondenza ecmpid
		{
			se ((demux [i]. ECMpids [j]. CAID == er-> caid || demux [i]. ECMpids [j]. CAID == er-> ocaid)
					&& Demux [i]. ECMpids [j]. ECM_PID == ehm> pid
					&& Demux [i]. ECMpids [j]. provid == ehm> prid
					&& Demux [i]. ECMpids [J]. VPID == er-> vpid)
				{ rompere ; }
		}
		se (j == demux [i]. ECMpidcount ) { continua ; }   // risposta ecm srvid ok ma no ecmpid corrispondente, forse questo per altro demuxer

		cs_log_dbg (D_DVBAPI, " Demuxer % d  % s controlword ricevuti per PID % d CAID % 04X PROVID % 06X ECMPID % 04X CHID % 04X VPID % 04X " , i,
					  (ER-> rc> = E_NOTFOUND? " no " : " " ), j, er-> caid, er-> prid, er-> pid, er-> chid, ehm> vpid);

		uint32_t status = dvbapi_check_ecm_delayed_delivery (i, er);

		uint32_t comparecw0 = 0 , comparecw1 = 0 ;
		char ecmd5 [ 17 * 3 ];
		cs_hexdump ( 0 , er-> ecmd5, 16 , ecmd5, sizeof (ecmd5));

		se (stato == 1 && er-> rc)    // ecmhash sbagliato
		{
			cs_log_dbg (D_DVBAPI, " Demuxer % d non è interessato a risposta ecmhash % s (richiesto uno diverso) " , i, ecmd5);
				continua ;
		}
		se (stato == 2 )    // nessun filtro
		{
			cs_log_dbg (D_DVBAPI, " Demuxer % d non è interessato a risposta ecmhash % s (filtro già ucciso) " , i, ecmd5);
			continua ;
		}
		se (stato == 5 )    // cw vuoto
		{
			cs_log_dbg (D_DVBAPI, " Demuxer % d non è interessato a risposta ecmhash % s (cw consegnato è vuota)! " , i, ecmd5);
			nocw_write = 1 ;
			se (ER-> rc <E_NOTFOUND) {er-> rc = E_NOTFOUND; }
		}

		se ((stato == 0 || stato == 3 || stato == 4 ) && er-> rc <E_NOTFOUND)    // 0 = corrispondenza ecm hash, 2 = nessun filtro, 3 = Reset tavolo, 4 = cache- ex risposta
		{
			se ( memcmp (er-> cw, demux [i]. lastcw [ 0 ], 8 ) == 0 && memcmp (er-> cw + 8 , demux [i]. lastcw [ 1 ], 8 ) == 0 )     // verifica per corrispondenza controlword
			{
				comparecw0 = 1 ;
			}
			altrimenti  se ( memcmp (er-> cw, demux [i]. lastcw [ 1 ], 8 ) == 0 && memcmp (er-> cw + 8 , demux [i]. lastcw [ 0 ], 8 ) == 0 )     // verificare la corrispondenza controlword
			{
				comparecw1 = 1 ;
			}
			se (== comparecw0 1 || comparecw1 == 1 )
			{
				cs_log_dbg (D_DVBAPI, " Demuxer % d duplicato controlword ecm risposta hash % s (duplicate controlword)! " , i, ecmd5);
				nocw_write = 1 ;
			}
		}

		se (stato == 3 )    // Reset tavolo
		{
			cs_log_dbg (D_DVBAPI, " Demuxer % d luckyshot nuovo controlword hash risposta ecm % s (ripristino tavolo ecm) " , i, ecmd5);
		}

		se (stato == 4 )    // nessun controllo sulle risposte di cache-ex!
		{
			cs_log_dbg (D_DVBAPI, " Demuxer % d nuovo controlword da lettore cache-ex (nessun controllo possibile ecmhash) " , i);
		}
		
		handled = 1 ; // marcare questa risposta ecm come gestito
		se (. ER-> rc <E_NOTFOUND && cfg dvbapi_requestmode == 0 && (demux [i]. pidindex == - 1 !) && er-> caid = 0 )
		{
			. demux [i] ECMpids . [j] tenta = 0xFE; // timeout di reset bandiera tentativi
			. demux [i] ECMpids [j]. irdeto_cycle = 0xFE; // resettare irdetocycle
			. demux [i] pidindex = j; // impostato indice corrente come * la * pid a decodificare
			demux [i]. ECMpids [j]. controllato = 4 ;
			cs_log_dbg (D_DVBAPI, " Demuxer % d ricomporre PID % d CAID % 04X PROVID % 06X ECMPID % 04X CHID % 02X VPID % 04X " ,
						  i, demux [i]. pidindex , er-> caid, er-> prid, er-> pid, er-> chid, ehm> vpid);
		}

		se (ER-> rc <E_NOTFOUND && cfg. dvbapi_requestmode == 1 && er-> caid! = 0 ) // TROVATO
		{
			pthread_mutex_lock (e demux [i]. answerlock ); // solo elaborare una risposta ecm
			se (demux [i]. ECMpids [j]. controllato ! = 4 )
			{

				int32_t t, o, ecmcounter = 0 ;

				per (t = 0 ; t <demux [i]. ECMpidcount ; t ++)   // controllare questo pid con controlword trovati per uno status più elevato:
				{
					se (t! = j && demux [i]. ECMpids [j]. stato > = demux [i]. ECMpids [t]. stato )
					{
# se ! WITH_STAPI definito &&! WITH_COOLAPI definito &&! WITH_MCA definito &&! WITH_AZBOX definito
						int32_t pidindex = demux [i]. pidindex ;
						se (pidindex == t) // controlla se inferiore pid stato già ricomporre!
						{ 
							int32_t . idx = demux [i] ECMpids . [j] indice . = demux [i] ECMpids . [pidindex] Indice ; // indice di swap con bassa pid stato
							demux [i]. ECMpids [pidindex]. Indice = 0 ; // resettare indice della vecchia pid!
							int32_t n;
							per (n = 0 ;. n <demux [i] STREAMpidcount ; n ++)
							{
								se (! demux [i]. ECMpids [j]. torrenti || demux [i]. ECMpids [J]. ruscelli & ( 1 << n))
								{
									dvbapi_set_pid (i, n, idx - 1 , true ); // consentire streampid utilizzato dalla nuova pid
								}
								altro
								{   
									dvbapi_set_pid (i, n, idx - 1 , falso ); // disabilita streampid non utilizzato dalle nuove pid  
								}
							}
							dvbapi_edit_channel_cache (i, pidindex, 0 ); // rimuovere lowerstatus pid da channelcache

						}
# endif
						. demux [i] ECMpids [t]. controllato = 4 ; // contrassegno di indice t come basso status

						per (o = 0 ; o <maxfilter; o ++)     // controllare se ecmfilter è in uso e arrestare tutti i ecmfilters di pid stato più basso
						{
							se (demux [i]. demux_fd [o]. fd > 0 && demux [i]. demux_fd [o]. Tipo == TYPE_ECM && demux [i]. demux_fd [o]. pidindex == t)
							{
								dvbapi_stop_filternum (i, o); // ecmfilter appartiene ad abbassare stato pid -> uccidere!
							}
						}
					}
				}
	

				per (o = 0 ; o <maxfilter; o ++) se (. demux [i] demux_fd . [o] tipo == TYPE_ECM) {ecmcounter ++; }    // contare tutte ecmfilters

				. demux [i] ECMpids . [j] tenta = 0xFE; // timeout di reset bandiera tentativi
				. demux [i] ECMpids [j]. irdeto_cycle = 0xFE; // resettare irdetocycle
				. demux [i] pidindex = j; // impostato indice corrente come * la * pid a decodificare

				se (ecmcounter == 1 )    // se totale ecmfilters esecuzione trovati è 1 -> abbiamo trovato il "migliore" pid
				{
					dvbapi_edit_channel_cache (i, j, 1 );
					. demux [i] ECMpids [j]. controllato = 4 ; // segnano migliore pid scorso;)
				}

				cs_log_dbg (D_DVBAPI, " Demuxer % d ricomporre PID % d CAID % 04X PROVID % 06X ECMPID % 04X CHID % 02X VPID % 04X " ,
					i, demux [i]. pidindex , er-> caid, er-> prid, er-> pid, er-> chid, ehm> vpid);
			}
			pthread_mutex_unlock (e demux [i]. answerlock ); // e rilasciarlo!
		}

		se (ER-> rc> = E_NOTFOUND)     // non trovato sul requestmode 0 + 1
		{
			se (ER-> rc == E_SLEEPING)
			{
				dvbapi_stop_descrambling (i);
				ritorno ;
			}
			
			struct s_dvbapi_priority * forceentry = dvbapi_check_prio_match (i, j, ' p ' );

			se (forceentry && forceentry-> forza)    // costretto pid? continuare a provare il ecmpid forzata!
			{
				se (! caid_is_irdeto (ER-> caid) || forceentry-> chid <0x10000)    // tutto cas o Irdeto cas con forzata chid prio
				{
					. demux [i] ECMpids . [j] tabella = 0 ;
					dvbapi_set_section_filter (i, er);
					continua ;
				}
				altro    cas // Irdeto senza chid prio costretti
				{
					se (demux [i]. ECMpids [j]. irdeto_curindex == 0xFE) {demux [i]. ECMpids [j]. irdeto_curindex = 0x00; }   // init irdeto indice corrente per primo
					se (! (demux [i]. ECMpids [j]. irdeto_curindex + 1 > demux [i]. ECMpids [j]. irdeto_maxindex ))   // verificare la presenza di ultima / max chid
					{
						cs_log_dbg (D_DVBAPI, " Demuxer % d cercando prossimo irdeto chid di FORZATA PID % d CAID % 04X PROVID % 06X ECMPID % 04X " , i,
									  j, er-> caid, er-> prid, ehm> pid);
						. demux [i] ECMpids [j]. irdeto_curindex ++; // indice irdeto uno su
						. demux [i] ECMpids . [j] tabella = 0 ;
						dvbapi_set_section_filter (i, er);
						continua ;
					}
				}
			}

			// In caso di timeout o evento LB fatale dare a questo pid un altro tentativo, ma non più di 1 prova
			se ((ER-> rc == E_TIMEOUT || (ER-> rcEx && er-> rcEx <= E2_CCCAM_NOCARD)) && demux [i]. ECMpids [j]. tenta == 0xFE)
			{
				. demux [i] ECMpids . [j] tenta = 1 ;
				. demux [i] ECMpids . [j] tabella = 0 ;
				dvbapi_set_section_filter (i, er);
				continua ;
			}
			altrimenti   non // tutti trovati risposte eccezione: prima risposta timeout e prima risposta loadbalancer fatale
			{
				demux [i]. ECMpids [j]. CHID = 0x10000; // sbarazzarsi di questo prio chid in quanto non avrebbe!
				. demux [i] ECMpids . [j] tenta = 0xFE; // ripristino timeout dei tentativi
			}

			se ( caid_is_irdeto (ER-> Caid))
			{
				se (demux [i]. ECMpids [j]. irdeto_curindex == 0xFE) {demux [i]. ECMpids [j]. irdeto_curindex = 0x00; }   // init irdeto indice corrente per primo
				se (! (demux [i]. ECMpids [j]. irdeto_curindex + 1 > demux [i]. ECMpids [j]. irdeto_maxindex ))   // verificare la presenza di ultima / max chid
				{
					cs_log_dbg (D_DVBAPI, " Demuxer % d cercando prossimo irdeto chid del PID % d CAID % 04X PROVID % 06X ECMPID % 04X VPID % 04X " , i,
								  j, er-> caid, er-> prid, er-> pid, ehm> vpid);
					. demux [i] ECMpids [j]. irdeto_curindex ++; // indice irdeto uno su
					. demux [i] ECMpids . [j] tabella = 0 ;
					dvbapi_set_section_filter (i, er);
					continua ;
				}
			}

			dvbapi_edit_channel_cache (i, j, 0 ); // rimuovere questo pid da channelcache
			se (demux [i]. pidindex == j)
			{
				demux [i]. pidindex = - 1 ; // pid attuale consegnato un notfound quindi questo pid isnt utilizzato per decodificare qualsiasi più lungo> chiaro pidindex
			}
			. demux [i] ECMpids [j]. irdeto_maxindex = 0 ;
			demux [i]. ECMpids [j]. irdeto_curindex = 0xFE;
			. demux [i] ECMpids . [j] tenta = 0xFE; // timeout di reset bandiera tentativi
			. demux [i] ECMpids [j]. irdeto_cycle = 0xFE; // resettare irdetocycle
			. demux [i] ECMpids . [j] tabella = 0 ;
			. demux [i] ECMpids [j]. controllato = 4 ; // ecmpid segnala come controllato
			. demux [i] ECMpids . [j] stato = - 1 ; // bandiera ecmpid come inutilizzabile
			int32_t trovato = 1 ; // impostazione per il primo run
			int32_t filternum = - 1 ;

			mentre (trovati> 0 )   // disattivare tutti i filtri ECM + emm per questo notfound
			{
				trovato = 0 ;
				filternum = dvbapi_get_filternum (i, ehm, TYPE_ECM); // ottenere ecm filternumber
				se (filternum> - 1 )    // in caso di filtri valido trovato
				{
					int32_t . fd = demux [i] demux_fd [filternum]. fd ;
					se (fd> 0 )   // in caso fd valido
					{
						dvbapi_stop_filternum (i, filternum); // fermata ecmfilter
						trovato = 1 ;
					}
				}
				se ( caid_is_irdeto (ER-> Caid))    // in caso irdeto cas fermano vecchi filtri emm
				{
					filternum = dvbapi_get_filternum (i, ehm, TYPE_EMM); // ottenere emm filternumber
					se (filternum> - 1 )    // in caso di filtri valido trovato
					{
						int32_t . fd = demux [i] demux_fd [filternum]. fd ;
						se (fd> 0 )   // in caso fd valido
						{
							dvbapi_stop_filternum (i, filternum); // fermata emmfilter
							trovato = 1 ;
						}
					}
				}
			}

			continua ;
		}


		// Sotto questo dovrebbe essere unica pista in caso di risposta ecm si trova

		uint32_t chid = get_subid (er); // traggono chid corrente in caso di Irdeto, o di una parte unica di ecm su altri sistemi CAS
		. demux [i] ECMpids . [j] CHID = (chid =! 0 chid: 0x10000?); // se non pari a zero applica, altrimenti utilizzare alcun valore chid 0x10000
		dvbapi_edit_channel_cache (i, j, 1 ); // farlo qui a qui dopo il CHID destra è registrato

		// Dvbapi_set_section_filter (i, er); non è più necessaria (sicuri)
		. demux [i] ECMpids . [j] tenta = 0xFE; // timeout di reset bandiera tentativi
		. demux [i] ECMpids [j]. irdeto_cycle = 0xFE; // resettare ciclo irdeto

		se (nocw_write || demux [i]. pidindex = j) { continua ; }   // cw è stato già scritto da un altro filtro o corrente pid isnt pid utilizzato per decodificare così finisce qui!

		struct s_dvbapi_priority * delayentry = dvbapi_check_prio_match (i, demux [i]. pidindex , ' d ' );
		se (delayentry)
		{
			se (delayentry-> ritardo < 1.000 )
			{
				cs_log_dbg (D_DVBAPI, " aspetta % d ms " , delayentry-> ritardo);
				cs_sleepms (delayentry-> ritardo);
			}
		}

		delayer (er);

		interruttore (selected_api)
		{
# ifdef WITH_STAPI
		caso STAPI:
			stapi_write_cw (i, er-> cw, demux [i]. STREAMpids , demux [i]. STREAMpidcount , demux [i]. pmt_file );
			rompere ;
# endif
		predefinito :
			dvbapi_write_cw (i, er-> cw, j);
			rompere ;
		}

		// Resettare idle-tempo
		client-> = ultima volta (( time_t *) 0 ); // ********* DA FISSO SEGUITO ******

		FILE * ecmtxt = NULL ;
		se (! cfg. dvbapi_listenport && cfg. dvbapi_boxtype ! = BOXTYPE_PC_NODMX)
			ecmtxt = fopen (ECMINFO_FILE, " w " );
		se (ecmtxt! = null && er-> rc <E_NOTFOUND)
		{
			char tmp [ 25 ];
			fprintf (ecmtxt, " caid: 0x ​​% 04X \ n pid: 0x ​​% 04X \ n prov: 0x % 06X \ n " , er-> caid, er-> pid, ( uint ) ER-> prid);
			interruttore (ER-> rc)
			{
			caso E_FOUND:
				se (ER-> selected_reader)
				{
					fprintf (ecmtxt, " lettore: % s \ n " , ehm> selected_reader-> etichetta);
					se ( is_network_reader (ER-> selected_reader))
						{ fprintf (ecmtxt, " da: % s \ n " , ehm> selected_reader-> dispositivo); }
					altro
						{ fprintf (ecmtxt, " da: locale \ n " ); }
					fprintf (ecmtxt, " il protocollo: % s \ n " , reader_get_type_desc (er-> selected_reader, 1 ));
					fprintf (ecmtxt, " luppolo: % d \ n " , ehm> selected_reader-> currenthops);
				}
				rompere ;

			caso E_CACHE1:
				fprintf (ecmtxt, " lettore: Cache \ n " );
				fprintf (ecmtxt, " da: cache1 \ n " );
				fprintf (ecmtxt, " protocollo: nessuno \ n " );
				rompere ;

			caso E_CACHE2:
				fprintf (ecmtxt, " lettore: Cache \ n " );
				fprintf (ecmtxt, " da: cache2 \ n " );
				fprintf (ecmtxt, " protocollo: nessuno \ n " );
				rompere ;

			caso E_CACHEEX:
				fprintf (ecmtxt, " lettore: Cache \ n " );
				fprintf (ecmtxt, " da: cache3 \ n " );
				fprintf (ecmtxt, " protocollo: nessuno \ n " );
				rompere ;
			}
			fprintf (ecmtxt, " tempo ecm: % .3f \ n " , ( float ) client-> cwlastresptime / 1000 );
			fprintf (ecmtxt, " cw0: % s \ n " , cs_hexdump ( 1 , demux [i]. lastcw [ 0 ], 8 , tmp, sizeof (tmp)));
			fprintf (ecmtxt, " CW1: % s \ n " , cs_hexdump ( 1 , demux [i]. lastcw [ 1 ], 8 , tmp, sizeof (tmp)));
		}
		se (ecmtxt)
		{
			int32_t ret = fclose (ecmtxt);
			se (ret < 0 ) { cs_log ( " ERRORE: Impossibile chiudere ecmtxt fd (errno = % d  % s ) " , errno, strerror (errno)); }
			ecmtxt = NULL ;
		}

	}
	se (gestito == 0 )
	{
		cs_log_dbg (D_DVBAPI, " la risposta non gestita ECM ricevuti per CAID % 04X PROVID % 06X ECMPID % 04X CHID % 04X VPID % 04X " ,
					  er-> caid, er-> prid, er-> pid, er-> chid, ehm> vpid);
	}

}

vuoto * dvbapi_start_handler ( struct s_client * cl, uchar * NON UTILIZZATO (mbuf), int32_t module_idx, void * (* _main_func) ( vuoto *))
{
	// Cs_log ("dvbapi caricato fd =% d", idx);
	se (cfg. dvbapi_enabled == 1 )
	{
		cl = create_client ( get_null_ip ());
		Cl> module_idx = module_idx;
		Cl> typ = ' c ' ;
		int32_t ret = pthread_create (e Cl> filo, NULL , _main_func, ( vuoto *) d);
		se (in pensione)
		{
			cs_log ( " ERRORE: Impossibile creare il thread handler dvbapi (errno = % d  % s ) " , ret, strerror (in pensione));
			ritorno  NULL ;
		}
		altro
			{ pthread_detach (Cl> filo); }
	}

	ritorno  NULL ;
}

vuoto * dvbapi_handler ( struct s_client * cl, uchar * mbuf, int32_t module_idx)
{
	tornare  dvbapi_start_handler (cl, mbuf, module_idx, dvbapi_main_local);
}

int32_t  dvbapi_set_section_filter ( int32_t demux_index, ECM_REQUEST * er)
{
	se (er) { ritorno - 1 ; }

	se (selected_api! = DVBAPI_3 && selected_api! = DVBAPI_1 && selected_api! = STAPI)    // valida solo per dvbapi3, dvbapi1 e STAPI
	{
		ritorno  0 ;
	}
	int32_t n = dvbapi_get_filternum (demux_index, ehm, TYPE_ECM);
	se (n < 0 ) { ritorno - 1 ; }   // in caso nessun filtro valida trovato;

	int32_t . fd = demux [demux_index] demux_fd [n]. fd ;
	se (fd < 1 ) { ritorno - 1 ; }   // in caso non fd valida

	uchar filtrare [ 16 ];
	maschera uchar [ 16 ];
	memset (filtro, 0 , 16 );
	memset (maschera, 0 , 16 );

	struct . s_ecmpids * curpid = & demux [demux_index] ECMpids [. demux [demux_index] demux_fd [n]. pidindex ];
	se (! curpid-> table = er-> ecm [ 0 ] && curpid-> table =! 0 ) { ritorno - 1 ; }   // se ecmtype corrente è diverso da ultimo richiesto ecmtype non si applicano sezione filtrante!
	uint8_t ecmfilter = 0 ;

	se (ER-> ecm [ 0 ] == 0x80) {ecmfilter = 0x81; }   // corrente ecm trattamento è ancora, la prossima sarà filtrata per dispari
	altro {ecmfilter = 0x80; } // corrente ecm trattamento è dispari, la prossima sarà filtrata anche per

	se (curpid-> tavolo! = 0 )    // ciclo ecmtype da dispari a pari o addirittura a dispari
	{
		filtrare [ 0 ] = ecmfilter; // solo accettare nuove ECM (se precedente dispari, filtro per anche e VisaVersa)
		maschera [ 0 ] = 0xFF;
		cs_log_dbg (D_DVBAPI, " Demuxer % d Filtra % d set ecmtable al % s (CAID % 04X PROVID % 06X FD % d ) " , demux_index, n + 1 ,
					  (Ecmfilter == 0x80? " EVEN " : " ODD " ), curpid-> CAID, curpid-> PROVID, fd);
	}
	altrimenti   // non decodifica in questo momento quindi stiamo interessavano tutti ecmtypes!
	{
		filtrare [ 0 ] = 0x80; // filtro impostato per attendere eventuali ECM
		maschera [ 0 ] = 0xF0;
		cs_log_dbg (D_DVBAPI, " Demuxer % d Filtra % d set ecmtable per ODD + EVEN (CAID % 04X PROVID % 06X FD % d ) " , demux_index, n + 1 ,
					  curpid-> CAID, curpid-> PROVID, fd);
	}
	uint32_t offset = 0 , extramask = 0xFF;
	uint32_t pid = demux [demux_index]. demux_fd [n]. pidindex ;

	struct s_dvbapi_priority * forceentry = dvbapi_check_prio_match (demux_index, pid, ' p ' );
	// Cs_log ("**** curpid-> CHID% 04X, controllato =% d, ehm> chid =% 04X *****", curpid-> CHID, curpid-> selezionato, ehm> chid) ;
	// Controllato 4 per assicurarsi che noi non impostare il filtro chid e nessuna di tali ECM in dvbstream tranne pid forzati!
	se (curpid-> CHID <0x10000 && (curpid-> verificata == 4 || (forceentry && forceentry-> forza)))
	{

		interruttore (ER-> caid >> 8 )
		{
		cassa 0x01:
			offset = 7 ;
			extramask = 0xF0;
			rompere ; // seca
		cassa 0x05:
			offset = 8 ;
			rompere ; // viaccess
		cassa 0x06:
			offset = 6 ;
			rompere ; // irdeto
		cassa 0x09:
			offset = 11 ;
			rompere ; // Videoguard
		caso 0x4A:   // DRE-Crypt, Bulcrypt, Tongang e gli altri?
			se (! caid_is_bulcrypt (ER-> Caid))
				{Offset = 6 ; }
			rompere ;
		}
	}

	int32_t irdetomatch = 1 ; // controllare se indice irdeto voluto è quello il trasporta chid corrente!
	se ( caid_is_irdeto (curpid-> CAID))
	{
		se (curpid-> irdeto_curindex == er-> ecm [ 4 ]) {irdetomatch = 1 ; }   // ok applicare il filtro chid
		altro {irdetomatch = 0 ; } // saltare filtraggio chid ma applicare irdeto filtraggio indice
	}

	se (offset && irdetomatch)   // abbiamo un cas con chid o parte unica in ecm controllato
	{
		i2b_buf ( 2 , curpid-> CHID, filtro + (offset - 2 ));
		maschera [(offset - 2 )] = 0xFF & extramask; // maschera aggiuntiva seca2 chid può essere FC10 o FD10 varia ogni mese, in modo che solo applicare F 10
		MASCHERA [(offset - 1 )] = 0xFF;
		cs_log_dbg (D_DVBAPI, " Demuxer % d Filtra % d set chid al % 04X su fd % d " , demux_index, n + 1 , curpid-> CHID, fd);
	}
	altro
	{
		se ( caid_is_irdeto (curpid-> CAID) && (curpid-> irdeto_curindex <0xFE))   // su irdeto possiamo sempre applicare irdeto filtraggio indice!
		{
			filtrare [ 2 ] = curpid-> irdeto_curindex;
			maschera [ 2 ] = 0xFF;
			cs_log_dbg (D_DVBAPI, " Demuxer % d Filtra % d set irdetoindex al % d su fd % d " , demux_index, n + 1 , curpid-> irdeto_curindex, fd);
		}
		altro   // tutti gli altri sistemi cas cas anche sistemi senza chid o parte unico ecm
		{
			cs_log_dbg (D_DVBAPI, " Demuxer % d Filtra % d set chid per QUALSIASI CHID su fd % d " , demux_index, n + 1 , fd);
		}
	}

	int32_t ret = dvbapi_activate_section_filter (demux_index, n, fd, curpid-> ECM_PID, filtro, maschera);
	se (ret < 0 )    // qualcosa è andato filtro impostazioni errate!
	{
		cs_log ( " Demuxer % d Filtro % d (fd % d !) errore di impostazione sezione filtrante -> Filtro fermata " , demux_index, n + 1 , fd);
		ret = dvbapi_stop_filternum (demux_index, n);
		se (ret == - 1 )
		{
			cs_log ( " Demuxer % d Filtro % d (fd % d ), l'arresto del filtro non riuscita -> uccidere tutti i filtri di questo demuxer! " , demux_index, n + 1 , fd);
			dvbapi_stop_filter (demux_index, TYPE_EMM);
			dvbapi_stop_filter (demux_index, TYPE_ECM);
		}
		ritorno - 1 ;
	}
	ritorno n;
}

int32_t  dvbapi_activate_section_filter ( int32_t demux_index, int32_t num, int32_t fd, int32_t pid, uchar * filtro, uchar * mask)
{

	int32_t ret = - 1 ;
	interruttore (selected_api)
	{
	caso DVBAPI_3:
	{
		struct dmx_sct_filter_params SFP2 ;
		memset (& SFP2 , 0 , sizeof ( SFP2 ));
		SFP2 . pid             = pid;
		SFP2 . timeout         = 0 ;
		SFP2 . flags           = DMX_IMMEDIATE_START;
		se (cfg. dvbapi_boxtype == BOXTYPE_NEUMO)
		{
			// DeepThought: sulle immagini DGS / CubeStation e NEUMO, forse altri
			// Il codice seguente è necessario per decodificare
			SFP2 . filtro . filtro [ 0 ] = Filtro [ 0 ];
			SFP2 . filtro . mascherare [ 0 ] = maschera [ 0 ];
			SFP2 . filtro . filtro [ 1 ] = 0 ;
			SFP2 . filtro . Maschera [ 1 ] = 0 ;
			SFP2 . filtro . filtro [ 2 ] = 0 ;
			SFP2 . filtro . maschera [ 2 ] = 0 ;
			memcpy ( SFP2 . filtro . filtrare + 3 , filtro + 1 , 16 - 3 );
			memcpy ( SFP2 . filtro . mascherare + 3 , maschera + 1 , 16 - 3 );
			// DeepThought: nei driver delle immagini DGS / CubeStation e Neumo,
			// Dvbapi 1 e 3 sono in qualche modo misto. Nei driver del kernel, il DMX_SET_FILTER
			// Ioctl si aspetta di ricevere una struttura dmx_sct_filter_params (dvbapi 3), ma
			// A causa di un bug relativi insiemi la "maschera positiva" a torto (che dovrebbero essere tutti 0).
			// D'altra parte, l'ioctl DMX_SET_FILTER1 utilizza anche i dmx_sct_filter_params
			// Struttura, che è errata (dovrebbe essere dmxSctFilterParams).
			// L'unico modo per farlo bene è quello di chiamare DMX_SET_FILTER1 con l'argomento
			// Atteso da DMX_SET_FILTER. In caso contrario, il parametro timeout non è passato correttamente.
			ret = dvbapi_ioctl (fd, DMX_SET_FILTER1, e SFP2 );
		}
		altro
		{
			memcpy ( SFP2 . filtro . filtro , il filtro, 16 );
			memcpy ( SFP2 . filtro . Maschera , la maschera, 16 );
			se (cfg. dvbapi_listenport || cfg. dvbapi_boxtype == BOXTYPE_PC_NODMX)
				ret = dvbapi_net_send (. DVBAPI_DMX_SET_FILTER, demux [demux_index] socket_fd , demux_index, num, ( unsigned  char *) e SFP2 );
			altro
				ret = dvbapi_ioctl (fd, DMX_SET_FILTER, e SFP2 );
		}
		rompere ;
	}

	caso DVBAPI_1:
	{
		struct dmxSctFilterParams SFP1 ;
		memset (& SFP1 , 0 , sizeof ( SFP1 ));
		SFP1 . pid = pid;
		SFP1 . timeout = 0 ;
		SFP1 . flags = DMX_IMMEDIATE_START;
		memcpy ( SFP1 . filtro . filtro , il filtro, 16 );
		memcpy ( SFP1 . filtro . Maschera , la maschera, 16 );
		ret = dvbapi_ioctl (fd, DMX_SET_FILTER1, e SFP1 );
		rompere ;
	}
# ifdef WITH_STAPI
	caso STAPI:
	{
		ret = stapi_activate_section_filter (fd, filtro, maschera);
		rompere ;
	}
# endif
	/ * # Ifdef WITH_COOLAPI NON ******* ancora implementato ********
	        caso COOLAPI: {
	            coolapi_set_filter (demux [demux_id] .demux_fd [n] .FD, n, pid, filtro, maschera, TYPE_ECM);
	            break;
	        }
	#endif
	* /
	predefinito :
		rompere ;
	}
	ritorno ret;
}


int32_t  dvbapi_check_ecm_delayed_delivery ( int32_t demux_index, ECM_REQUEST * er)
{
	char nullcw [CS_ECMSTORESIZE];
	memset (nullcw, 0 , CS_ECMSTORESIZE);
	se ( memcmp (ER-> cw, nullcw, 8 ) == 0 && memcmp (ER-> cw + 8 , nullcw, 8 ) == 0 ) { restituire  5 ;} // ricevuto un cw null -> non utilizzabile!
	int32_t filternum = dvbapi_get_filternum (demux_index, ehm, TYPE_ECM);
	se (filternum < 0 ) { ritorno  2 ; }   // se nessun atto filtro corrispondente come risposta ecm è in ritardo
	struct s_ecmpids * curpid = & demux [demux_index]. ECMpids [demux [demux_index]. demux_fd [filternum]. pidindex ];
	se (curpid-> Tavolo == 0 ) { ritorno  3 ; }   // il cambiamento si trova tavolo agire come risposta ecm
	se (ER-> rc == E_CACHEEX) { ritorno  4 ; }   // il cache-ex si trova atto risposta come risposta ecm
	
	se ( memcmp (demux [demux_index]. demux_fd [filternum]. ecmd5 , nullcw, CS_ECMSTORESIZE))
	{
		char ecmd5 [ 17 * 3 ];
		cs_hexdump ( 0 ., demux [demux_index] demux_fd [filternum]. ecmd5 , 16 , ecmd5, sizeof (ecmd5));
		cs_log_dbg (D_DVBAPI, " Demuxer % d ha chiesto controlword per ecm % s su fd % d " , demux_index, ecmd5, demux [demux_index]. demux_fd [filternum]. fd );
		ritorno  memcmp (. demux [demux_index] demux_fd [filternum]. ecmd5 , er-> ecmd5, CS_ECMSTORESIZE); // 1 = nessuna risposta sul ecm chiediamo scorso per questo fd!
	}
	altro { ritorno  0 ; }
}

int32_t  dvbapi_get_filternum ( int32_t demux_index, ECM_REQUEST * ehm, int32_t tipo)
{
	se (er) { ritorno - 1 ; }

	int32_t n;
	int32_t fd = - 1 ;

	per (n = 0 ; n <maxfilter; n ++)     // determinare fd
	{
		se (demux [demux_index]. demux_fd [n]. fd > 0 && demux [demux_index]. demux_fd [n]. tipo == tipo)      // verifica per tipo valido e destra (ECM o EMM)
		{
			se ((demux [demux_index]. demux_fd [n]. pid == er-> pid) &&
					((Demux [demux_index]. demux_fd [n]. provid == er-> prid) || demux [demux_index]. demux_fd [n]. provid == 0 ) &&
					((Demux [demux_index]. demux_fd [n]. caid == er-> Caid) || (demux [demux_index]. demux_fd [n]. caid == er-> ocaid))) // corrente pid ecm?
			{
				. fd = demux [demux_index] demux_fd [n]. fd ; // trovato!
				rompere ;
			}
		}
	}
	se (fd> 0 && demux [demux_index]. demux_fd [n]. provid == 0 ) {demux [demux_index]. demux_fd [n]. provid = er-> prid; }   // incidere compilare provid in demuxer

	ritorno (fd> 0 n: fd); // restituisce -1 (fd) sulla non trovato, su trovata filternumber ritorno (n)
}

int32_t  dvbapi_ca_setpid ( int32_t demux_index, int32_t pid)
{
	int32_t idx = - 1 , n;
	per (n = 0 ;. n <demux [demux_index] ECMpidcount ; n ++)   (! = no decodifica possibile) // cleanout vecchi indici di PID che hanno ora Statuto ignorare
	{
		idx = demux [demux_index]. ECMpids [n]. Indice ;
		se (! IDX) continuare ; // saltare ecmpids che non vengono utilizzati per decifrare
		se (demux [demux_index]. ECMpids [n]. stato == - 1 . || demux [demux_index] ECMpids . [n] controllati == 0 ) { // resettare indice!
			. demux [demux_index] ECMpids . [n] index = 0 ;
			int32_t i;
			per (i = 0 ; i <DEMUX [demux_index]. STREAMpidcount && IDX; i ++) {
				dvbapi_set_pid (demux_index, i, idx - 1 , falso ); // disabilitare tutti streampids per questo indice!
			}
		}
	}

	. idx = demux [demux_index] ECMpids . [pid] indice ;

	se (! IDX)    // se non indicizzatore per questo Pid ottenere uno!
	{
		idx = dvbapi_get_descindex (demux_index);
		. demux [demux_index] ECMpids . [pid] index = idx;
		cs_log_dbg (D_DVBAPI, " Demuxer % d PID: % d CAID: % 04X ECMPID: % 04X utilizza indice % d " , demux_index, pid,
					  . demux [demux_index] ECMpids . [pid] CAID ., demux [demux_index] ECMpids [pid]. ECM_PID , idx - 1 );
	}

	per (n = 0 ;. n <demux [demux_index] STREAMpidcount ; n ++)
	{
		se (! demux [demux_index]. ECMpids [pid]. torrenti || demux [demux_index]. ECMpids [pid]. flussi & ( 1 << n)) {
			dvbapi_set_pid (demux_index, n, idx - 1 , true ); // consentono streampid
		}
		altro {
			dvbapi_set_pid (demux_index, n, idx - 1 , falso ); // disabilita streampid
		} 
	}

	sul rimpatrio idx - 1 ; // restituisce caindexer
}

int8_t  update_streampid_list ( uint8_t cadevice, uint16_t pid, int32_t idx)
{
	struct s_streampid * listitem, * newlistitem;
	se (! ll_activestreampids)
		{Ll_activestreampids = ll_create ( " ll_activestreampids " ); }
	LL_ITER itr;
	se ( ll_count (ll_activestreampids)> 0 )
	{
		itr = ll_iter_create (ll_activestreampids);
		mentre ((listitem = ll_iter_next (& ITR)))
		{
			se (cadevice == listitem-> cadevice && pid == listitem-> streampid) {
				se (listitem-> activeindexers & ( 1 << IDX)) {
					tornare FOUND_STREAMPID_INDEX; // risultato trovato
				} altro {
					listitem-> activeindexers | = ( 1 << IDX); // ca + pid trovato ma non questo indice -> aggiungere l'indice
					cs_log_dbg (D_DVBAPI, " Aggiunto streampid esistente % 04X con il nuovo indice % d per circa % d " , pid, idx, cadevice);
					ritorno ADDED_STREAMPID_INDEX;
				}
			}
		}
	}
	se (! cs_malloc (& newlistitem, sizeof ( struct s_streampid)))
		{ ritorno ADDED_STREAMPID_INDEX; }
	newlistitem-> cadevice = cadevice;
	newlistitem-> streampid = pid;
	newlistitem-> activeindexers = ( 1 << idx);
	ll_append (ll_activestreampids, newlistitem);
	cs_log_dbg (D_DVBAPI, " Aggiunto nuovo streampid % 04X con indice % d per circa % d " , pid, idx, cadevice);
	ritorno ADDED_STREAMPID_INDEX;
}

int8_t  remove_streampid_from_list ( uint8_t cadevice, uint16_t pid, int32_t idx)
{
	se (ll_activestreampids!) restituiscono NO_STREAMPID_LISTED;
	
	struct s_streampid * listitem;
	
	LL_ITER itr;
	se ( ll_count (ll_activestreampids)> 0 )
	{
		itr = ll_iter_create (ll_activestreampids);
		mentre ((listitem = ll_iter_next (& ITR)))
		{
			se (cadevice == listitem-> cadevice && pid == listitem-> streampid) {
				se (idx = - 1 && listitem-> activeindexers & ( 1 << IDX)) {
					listitem-> activeindexers & = ~ ( 1 << IDX); // bandiera come disabilitata per questo indice
				}
				se (idx == - 1 ) { // IDX -1 significa disattivare tutti!
					listitem-> activeindexers = 0 ;
				}
				cs_log_dbg (D_DVBAPI, " Togli streampid % 04X usando indicizzatore % d da ca % d " , pid, idx, cadevice);
				se (listitem-> activeindexers == 0 ) { // tutte indicizzatori disabilitati? -> Rimuovere pid dalla lista!
					ll_iter_remove_data (& ITR);
					cs_log_dbg (D_DVBAPI, " Rimosso ultima indicizzatore di streampid % 04X da ca % d " , pid, cadevice);
					ritorno REMOVED_STREAMPID_LASTINDEX;
				}
				ritorno REMOVED_STREAMPID_INDEX;
			}
		}
	}
	ritorno NO_STREAMPID_LISTED;
}

vuoto  disable_unused_streampids ( int16_t demux_id)
{
	se (ll_activestreampids!) di ritorno ;
	se ( ll_count (ll_activestreampids) == 0 ) ritorno ; // elementi in lista?
	
	int32_t ecmpid = demux [demux_id]. pidindex ;
	se (== ecmpid - 1 ) ritorno ; // no ecmpid attivo!
	
	int32_t idx = demux [demux_id]. ECMpids [ecmpid]. Indice ;
	int32_t i, n;
	struct s_streampid * listitem;
	// Ricerca di vecchie streampids abilitati su tutti i dispositivi ca che devono essere disattivato, indice 0 viene saltato in quanto appartiene a FTA!
	per (i = 0 ; i <MAX_DEMUX && idx; i ++) {
		se ((demux [demux_id]!. ca_mask & ( 1 << i))) proseguire ; // continuare se ca è inutilizzato da questo demuxer
		
		LL_ITER itr;
		itr = ll_iter_create (ll_activestreampids);
		mentre ((listitem = ll_iter_next (& ITR)))
		{
			se (! i = listitem-> cadevice) continuare ; // ca doesnt partita
			se ((listitem-> activeindexers & (! 1 << (idx- 1 )))) continua ; // indice doesnt partita
			per (n = 0 ;. n <demux [demux_id] STREAMpidcount ; n ++) {
				se (listitem-> streampid == demux [demux_id]. STREAMpids [n]) { // verificare se le partite PID con streampid corrente demuxer
					rompere ;
				}
			}
			se (n == demux [demux_id]. STREAMpidcount ) {
				. demux [demux_id] STREAMpids [n] = listitem-> streampid; // messo temperatura qui!
				dvbapi_set_pid (demux_id, n, idx - 1 , falso ); // Non sono stati trovati così disabilitare questa streampid ora inutilizzato
				. demux [demux_id] STREAMpids [n] = 0 ; // rimuovere temperatura!
			}
		}
	}
}


int8_t  is_ca_used ( uint8_t cadevice)
{
	se (!) ll_activestreampids ritorno CA_IS_CLEAR;
	
	struct s_streampid * listitem;
	
	LL_ITER itr;
	se ( ll_count (ll_activestreampids)> 0 )
	{
		itr = ll_iter_create (ll_activestreampids);
		mentre ((listitem = ll_iter_next (& ITR)))
		{
			se (! listitem-> cadevice = cadevice) continua ;
			ritorno CA_IS_IN_USE;
		}
	}
	ritorno CA_IS_CLEAR;
}

const  char * dvbapi_get_client_name ( vuoto )
{
	ritorno client_name;
}

vuoto  check_add_emmpid ( int32_t demux_index, uchar * filtro, int32_t l, int32_t emmtype)
{
	se (l < 0 ) di ritorno ;
	
	uint32_t typtext_idx = 0 ;
	int32_t ret = - 1 ;
	const  char * typtext [] = { " UNICA " , " CONDIVISA " , " GLOBAL " , " UNKNOWN " };

	mentre (((emmtype >> typtext_idx) e 0x01) == 0 && typtext_idx < sizeof (typtext) / sizeof ( const  char *))
	{
		++ Typtext_idx;
	}
	
	// Filtro già in lista?
	se ( is_emmfilter_in_list (filtro, demux [demux_index]. EMMpids [l]. PID , demux [demux_index]. EMMpids [l]. PROVID , demux [demux_index]. EMMpids [l]. CAID ))
	{
		cs_log_dbg (D_DVBAPI, " Demuxer % d duplicato emm tipo di filtro % s , emmpid: 0x ​​% 04X , emmcaid: % 04X , emmprovid: % 06X -> SALTATA! " , demux_index,
			typtext [typtext_idx], demux [demux_index]. EMMpids [l]. PID , demux [demux_index]. EMMpids [l]. CAID , demux [demux_index]. EMMpids [l]. PROVID );
		ritorno ;
	}

	se (demux [demux_index]. emm_filter > = demux [demux_index]. max_emm_filter ) // può essere avviato questo filtro? se non aggiungere alla lista dei emmfilters inattive
	{
		add_emmfilter_to_list (demux_index, filtro, demux [demux_index]. EMMpids [l]. CAID , demux [demux_index]. EMMpids [l]. PROVID , demux [demux_index]. EMMpids [l]. PID , 0 , falso );
	}
	altro  // attivare questa emmfilter
	{
		ret = dvbapi_set_filter (demux_index, selected_api, demux [demux_index]. EMMpids [l]. PID , demux [demux_index]. EMMpids [l]. CAID ,
			. demux [demux_index] EMMpids . [l] PROVID , il filtro, il filtro + 16 , 0 , demux [demux_index]. pidindex , TYPE_EMM, 1 );
	}
	se (ret = - 1 )
	{
		se (. demux [demux_index] emm_filter == - 1 ) // innanzitutto eseguire -1
		{
			. demux [demux_index] emm_filter = 0 ;
		}
		demux [demux_index]. emm_filter ++; // incremento complessivo filtri attivi
		cs_log_dump_dbg (D_DVBAPI, filtro, 32 , " Demuxer % d iniziato emm tipo di filtro % s , pid: 0x ​​% 04X " , demux_index, typtext [typtext_idx], demux [demux_index]. EMMpids [l]. PID );
		ritorno ;
	}
	altrimenti    // non impostato successo, quindi aggiungerlo alla lista per riprovare più tardi!
	{
		add_emmfilter_to_list (demux_index, filtro, demux [demux_index]. EMMpids [l]. CAID , demux [demux_index]. EMMpids [l]. PROVID , demux [demux_index]. EMMpids [l]. PID , 0 , falso );
		cs_log_dump_dbg (D_DVBAPI, filtro, 32 , " Demuxer % d aggiunto inattivo emm tipo di filtro % s , pid: 0x ​​% 04X " , demux_index, typtext [typtext_idx], demux [demux_index]. EMMpids [l]. PID );
	}
	ritorno ;
}

uint16_t  dvbapi_get_client_proto_version ( vuoto )
{
	ritorno client_proto_version;
}

/ *
 * Struttura protocollo
 * /

vuoto  module_dvbapi ( struct s_module ph *)
{
	pH> desc = " dvbapi " ;
	pH> type = MOD_CONN_SERIAL;
	pH> listenertype = LIS_DVBAPI;
# se definito (WITH_AZBOX)
	pH> s_handler = azbox_handler;
	pH> send_dcw = azbox_send_dcw;
# elif definito (WITH_MCA)
	pH> s_handler = mca_handler;
	pH> send_dcw = mca_send_dcw;
	selected_box = selected_api = 0 ; // HACK: Questo risolve avviso non corretto su fuori dai limiti di accesso agli array in functionas che non sono nemmeno chiamati quando WITH_MCA è definito
# altro
	pH> s_handler = dvbapi_handler;
	pH> send_dcw = dvbapi_send_dcw;
# endif
}
#endif // HAVE_DVBAPI
